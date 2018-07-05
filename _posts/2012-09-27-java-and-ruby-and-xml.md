---
layout: post
title: "Java and Ruby and XML"
date: 2012-09-27 08:07
comments: false
categories: [java, ruby, xml]
---

I've been working with a colleague on a project to create a set of command line applications to read and emit various genomics file formats. The clients asked that we write the applications in Java, since we'll be handing it over to them and it's the language they're most comfortable with.

One of the file formats is [mzidentML](http://www.psidev.info/mzidentml), an XML format.

It's been a while since I've used Java in anger, and using it to read XML probably longer, so it was virtually like learning it anew.

Even with my more Java adept colleague, it took hours to work out how to use the Java XML API for this relatively simple task. So by noting the analogous operations using Ruby and [Nokogiri](http://nokogiri.org/), I'll save myself the grief in future.

It's also interesting to note where I think things in the Java XML API are more verbose or complicated, but probably some of the myriad of external Java XML processing packages address these.

## Namespaces ##

If your XML uses namespaces, like mzidentML does:

    <mzIdentML id="" version="1.0.0"
         xsi:schemaLocation="http://psidev.info/psi/pi/mzIdentML/1.0 http://mascotx/mascot/xmlns/schema/mzIdentML/mzIdentML1.0.0.xsd"
         xmlns="http://psidev.info/psi/pi/mzIdentML/1.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         creationDate="2012-09-14T11:21:00">

then you need to specify the namespace in your XPath expressions.

Here is the minimum required (as best as I could determine) to set an XML namespace:

    xPath.setNamespaceContext(new NamespaceContext() {
        public String getNamespaceURI(String prefix) {
            if (prefix == null)
                throw new NullPointerException("Null prefix");
            else if ("mz".equals(prefix))
                return "http://psidev.info/psi/pi/mzIdentML/1.0";
            return XMLConstants.NULL_NS_URI;
        }
        public String getPrefix(String uri) {
            throw new UnsupportedOperationException();
        }
        public Iterator getPrefixes(String uri) {
            throw new UnsupportedOperationException();
        }
    });

So that it can be used in an XPath expression as so:

    NodeList peptideList = (NodeList)xPath.evaluate("//mz:Peptide", rootNode, XPathConstants.NODESET);

In Ruby, using Nokogiri, it can basically be:

    MZIDENTML_NS = 'http://psidev.info/psi/pi/mzIdentML/1.0';
    # Look for <Peptide> elements in this namespace
    peptide_list = doc.xpath("//mz:Peptide", {'mz' =>  MZIDENTML_NS})

Now I can appreciate somewhat that the Java NamespaceContext interface is trying to cover more than this simple use case. However I feel there should be, in the core Java API, a simple alternative specifically for this common use case.

## XPath Expressions ##

There was a hint of these in the section above.

To evaluate an XPath expression in Java:

    NodeList peptideList = (NodeList)xPath.evaluate("//mz:Peptide", rootNode, XPathConstants.NODESET);

The casting and the constant are understandable in a statically typed language, although I did wonder if method overloading could be used to avoid the casting (obviously the last parameter could not be a String constant, perhaps NodeList.class for example).

The Ruby/Nokogiri version of searching for elements with XPath:

    peptide_list = doc.xpath("//mz:Peptide", {'mz' =>  MZIDENTML_NS})

## Retrieving Attributes ##

Given the following XML:

    <PeptideEvidence id="PE_11_1_KPYK1_YEAST_0_469_474" start="469" end="474" pre="K" post="E" missedCleavages="0" isDecoy="false" DBSequence_Ref="DBSeq_1_KPYK1_YEAST" />

It took a while to work out a way to get an XML attribute value with the Java XML API. I was convinced this must not be right, but it seems that it is:

    String id = node.getAttributes().getNamedItem("id").getNodeValue();

Contrast that with Ruby/Nokogiri:

    id = node['id']

Completely straightforward and intuitive.

## Conclusion ##

I'm sure we could have cast around and evaluated the available XML parsing/handling libraries for Java. Many probably make life a lot simpler.

Ultimately we didn't because we thought the task was simple and straightforward, and hence didn't need another potentially large library. We thought the provided Java XML API should be sufficient. It was, but it wasn't straightforward.
