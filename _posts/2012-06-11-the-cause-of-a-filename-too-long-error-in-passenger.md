---
layout: post
title: "The Cause of a Filename too long error in Passenger"
date: 2012-06-11 21:16
comments: false
categories: [rails, passenger, troubleshooting]
---

Due to a space issue on a server, I moved the location I had deployed a Rails app to another mount point. This is all easy, just move the files, check permissions, and make a few path changes in the Apache configuration file for that virtualhost.

But when I did a reload on Apache, the app didn't come back up.

I headed straight for the appropriate Apache error log to see if there was anything enlightening. What looked interesting was:

    [ pid=14748 thr=3086636752 file=ext/apache2/HelperAgent.cpp:353 time=2012-06-04 15:38:41.247 ]: Cannot create Unix socket '/spare/[path redacted for this blog]/shared/system/passenger.1.0.17475/generation-11/spawn-server/socket.14748.139057232': filename is too long.

What made no sense to me was that the quoted path could be too long for Linux.

I tried my luck searching the web and the best I got was [an issue raised on 24th May 2011](http://code.google.com/p/phusion-passenger/issues/detail?id=674), which was unresolved<sup>[1](#footnote)</sup>.

Guess I was going to have to do this the hard way.

The upside was that the error message had a stack trace that pointed to a root cause.

     in 'void Passenger::SpawnManager::restartServer()' (SpawnManager.h:170)
     in 'Passenger::SpawnManager::SpawnManager(const std::string&, const boost::shared_ptr<Passenger::ServerInstanceDir::Generation>&, const Passenger::AccountsDatabasePtr&, const std::string&, const Passenger::AnalyticsLoggerPtr&, int, const std::string&)' (SpawnManager.h:535)
     in 'Passenger::ApplicationPool::Pool::Pool(const std::string&, const boost::shared_ptr<Passenger::ServerInstanceDir::Generation>&, const Passenger::AccountsDatabasePtr&, const std::string&, const Passenger::AnalyticsLoggerPtr&, int, const std::string&)' (Pool.h:1078)
     in 'Server::Server(Passenger::FileDescriptor, pid_t, const std::string&, bool, const std::string&, const std::string&, const std::string&, const std::string&, unsigned int, unsigned int, unsigned int, unsigned int, const Passenger::VariantMap&)' (HelperAgent.cpp:239)
     in 'int main(int, char**)' (HelperAgent.cpp:343)

I downloaded the Phusion Passenger source and starting looking. Line 170 of SpawnManager.h was a trace breakpoint, but directly below it was:

    socketFilename = generation->getPath() + "/spawn-server/socket." +
        toString(getpid()) + "." +
        pointerToIntString(this);
    socketPassword = Base64::encode(random.generateByteString(32));
    serverSocket = createUnixServer(socketFilename.c_str());

So I could see the construction of the full path for the socket file, and that was being passed to createUnixServer(). Wielding [ack](http://betterthangrep.com/), I located createUnixServer in passenger/ext/common/Utils/IOUtils.cpp and found this interesting check:

    if (filename.size() > sizeof(addr.sun_path) - 1) {
        string message = "Cannot create Unix socket '";
        message.append(filename.toString());
        message.append("': filename is too long.");
        throw RuntimeException(message);
    }

Hmmm, the effective path length of the socket filename is limited by `sizeof(addr.sun_path)`. The `addr` variable is of type `struct sockaddr_un`. With some more web searching [I found the definition for this type](http://www.ccplusplus.com/2011/10/struct-sockaddrun.html):

    struct sockaddr_un {
        unsigned short sun_family;  /* AF_UNIX */
        char sun_path[108];
    }

So the limit of the socket filename in Passenger (full path) is 107 chars (108 - 1). When I moved my webapp I changed the PassengerTempDir path value in the Apache configuration accordingly. The new path (plus the socketFilename auto-generated path) exceeded 107 chars. Augh.

This effectively means that, with the minimum auto-generated socket filename path being _around_ 70 chars, you're left with _about_ 37 chars to play with for the PassengerTempDir path.

I adjusted my paths accordingly, and I could fire up the Rails app successfully.

As to why the sockaddr_un struct has that limit? I haven't been able to find out yet.

---
<a id="footnote"></a>
1. I've now added my findings as a comment.
