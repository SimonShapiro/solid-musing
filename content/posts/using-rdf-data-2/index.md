---
title: "Using RDF data from other apps - 2"
date: 2019-03-13T07:51:03Z
draft: true
---
In the [last note](../using-rdf-data-1), I identified a number of issues or challenges to using rdf.  In particular I pointed to three that I plan to experiment with here:

- How to have a general purpose application with a more human friendly UI?
- How do we bridge the divide between document orientation and the store orientation?
- How do applications signal the data they expect or are creating?

In order to progress this I have created a 'test bed' using node.js.  The 'test bed' consists of the barest minimum lines of code to get data into an rdflibjs store.  Its purpose is to give us the ability to concentrate on the rdf pieces necessary to answer the questions, rather than to build a complete application.

# 1. Setting up the 'test bed'

When I am learning something new in technology, I try to de-clutter my thinking as much as possible by starting with very few lines of code that I understand instead of going immediately into a new framework or group of libraries.  In this case, I want to use rdflibjs, but without all the valuable server and security functions of a solid server.  So let's get started.

I mentioned in the previous article that to me the turtle format was as straightforward as a yaml file. So below is a simple config file written as turtle.

```
@prefix : <#> .
@prefix dc: <http://purl.org/dc/terms/> .

<> a :ConfigFile ;
    dc:title "A config file" .

:config a :Configuration;
    :path "/PATH_TO_DATA";
    :inputFile "/data/bookmarks.ttl";
    :outputFile "/data/results.ttl" .
```

The last four lines are fairly typical in yaml as well.  They set up three values one each for, a `path`, an `inputFile` name, and an `outputFile` name.  We also tie them together under the 'heading' `:config`.  From here it is necessary to start delving into turtle syntax.

## 1.1 What's in a name?

Some of the best practices for rdf and turtle are in the [linked data book](http://linkeddatabook.com/editions/1.0/).  In it the authors make the point that a URI can be thought of as the unique **name** of the document.  When this name is resolved by the web, we retrieve a representation of it in turtle format.

So the lines 
```
<> a :ConfigFile ;
    dc:title "A config file" .
```
are telling us: "this document is a configFile with the title 'A config file'".  Also, our understanding of ConfigFile is limited to this document as indicated by `:`.  Remember, `<> a :ConfigFile` is turned into 
```
<URI> <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <URI#ConfigFile>
```
internally, where `URI` is the location of the document.

## 1.2 Being server free

Earlier I said that we wanted a set-up that had a minimal amount of lines of code and fuss. Also, I want to run this on nodejs rather than a browser. 

There were three options for locating this document: on the file system, on the file system with a simple web-server, or on the file system with a solid-server.  I chose the first option for the experiment.

Here is the code that I use to get data into the rdflibjs store:

```
const fetchFromFs = async( fileName ) => {
    return new Promise ( (resolve, reject ) => {
        fs.readFile(fileName, (err, data) => {
            if (err) reject(err);
            resolve(data.toString('utf-8'))
          });
    })
}

const createStoreFromFile = async (file) => {
    return new Promise( async (resolve, reject) => {
        console.log("fetching from", file)
        const store = $rdf.graph()
        const data = await fetchFromFs(file)
        try {
            $rdf.parse(data, store, "file://"+file, 'text/turtle')
            resolve(store)
        }
        catch (e){
            reject("Error parsing data")
        }
    })
}
```

For completeness, here is a minimal function to get the data into the store from a server using the fetch API:

```
const createStoreFromUri = async (uri) => {
    return new Promise( async (resolve, reject) => {
        const store = $rdf.graph()
        const result = await fetch(uri)
        const data = await result.text()
        try {
            $rdf.parse(data, store, uri, 'text/turtle')
            resolve(store)
        }
        catch (e){
            reject("Error parsing data")
        }
    })
}
```

These functions should **not** be used in production, they are only here for illustrative purposes.  For production on solid servers you should choses functions provided within the solid ecosystem.  These will help you with authentication, parsing, and error detection.