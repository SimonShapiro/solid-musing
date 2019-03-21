---
title: "Using RDF data from other apps - 2"
date: 2019-03-21T17:00:03Z
draft: false
---
In the [last note](../using-rdf-data-1), I identified a number of issues or challenges to using rdf.  In particular I pointed to three that I plan to experiment with here:

- How to have a general purpose application with a more human friendly UI?
- How do we bridge the divide between document orientation and the store orientation?
- How do applications signal the data they expect or are creating?

In order to progress this I have created a 'test bed' using node.js.  The 'test bed' consists of the barest minimum lines of code to get data into an `rdflibjs` store.  Its purpose is to give us the ability to concentrate on the rdf pieces necessary to answer the questions, rather than to build a complete application.

# 1. Setting up the 'test bed'

When I am learning something new in technology, I try to de-clutter my thinking as much as possible by starting with very few lines of code that I understand instead of going immediately into a new framework or group of libraries.  In this case, I want to use `rdflibjs`, but without all the valuable server and security functions of a solid server.  

Also, the sample data has been extracted from a solid app.  This section will only read the data files and will **not** write anything back to solid.  We are trying to isolate our understanding of `rdflibjs`.  So let's get started.

## 1.1 Being server free

Earlier I said that we wanted a set-up that had a minimal amount of lines of code and fuss. Also, I want to run this on nodejs rather than a browser. 

There were three options for locating this document: on the file system, on the file system with a simple web-server, or on the file system with a solid-server.  I chose the first option for the experiment.

Here is the code that I use to get data into the `rdflibjs` store:

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

## 1.2 Using the test bed.

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

The code below shows how to use the functions to retrieve the `inputFile` from the config file.

```
( async () => {
    const configFile = "/PATH_TO_DATA/config.ttl"
    const store2 = await createStoreFromFile(configFile)
    const CONFIG = $rdf.Namespace("file://"+configFile+"#")
    const path = store2.any(CONFIG("config"), CONFIG("path"), undefined, undefined)
    const inputFile = store2.any(CONFIG("config"), CONFIG("inputFile"), undefined, undefined)
    console.log(path.value+inputFile.value)
})();

```

## 1.3 What's happening?

The last four lines of the config file are fairly typical in yaml as well.  They set up three values one each for, a `path`, an `inputFile` name, and an `outputFile` name.  We also tie them together under the 'heading' `:config`.  From here it is necessary to start delving into turtle syntax.

Some of the best practices for rdf and turtle are in the [linked data book](http://linkeddatabook.com/editions/1.0/).  In it the authors make the point that a URI can be thought of as the unique **name** of the document.  When this name is resolved by the web, we retrieve a representation of it in turtle format.

So the lines 
```
<> a :ConfigFile ;
    dc:title "A config file" .
```
are telling us: "this document is a configFile with the title 'A config file'".  Also, our understanding of ConfigFile is limited to this document as indicated by `':'`.  Remember, `<> a :ConfigFile` is turned into 
```
<URI> <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <URI#ConfigFile>
```
internally, where `URI` is the location of the document.  In this case, because we have chosen to use the file system, the document URI is `file:///PATH_TO_DATA/config.ttl`.

The line 
```
    const CONFIG = $rdf.Namespace("file://"+configFile+"#")
```
simply sets up a short-hand function for the namespace.  It is the `rdflibjs` equivalent of `@prefix`.

The store function `.any` performs pattern matching on the rdf-triple.  Actually, `rdflibjs`, is a quad store which also has a fourth value (known in the community as *why*) which usually contains the URI of the original document.

# 2. Summary

So now we have the basic setup that lets us utilise `rdflibjs` from `nodejs` and taking the files from the node file system.

A typical pattern of usage in solid apps is to find a `settings` file and then use the information there to find the location of rdf data and retrieve it.  Without using the solid libraries and a solid server, we will illustrate this process using our simple setup and the config file.

```
( async () => {
    const store2 = await createStoreFromFile(configFile)
    const CONFIG = $rdf.Namespace("file://"+configFile+"#")
    const path = store2.any(CONFIG("config"), CONFIG("path"), undefined, undefined)
    const inputFile = store2.any(CONFIG("config"), CONFIG("inputFile"), undefined, undefined)
    console.log(path.value+inputFile.value)
    const bookmarksStore = await createStoreFromFile(path.value+inputFile.value)
    bookmarksStore.match($rdf.sym("http://www.w3.org/2002/01/bookmark#Bookmark"),undefined, undefined, undefined).forEach(statement => {
        console.log(statement.subject.value, statement.predicate.value, statement.object.value)
    })
})();
```

This short example retrieves the location inputFile from the config file and then retrieves information about the bookmark contained in inputFile.  The output is below:

```
file:///PATH_TO_DATA/config.ttl#0.056127920957769306 http://www.w3.org/1999/02/22-rdf-syntax-ns#type http://www.w3.org/2002/01/bookmark#Bookmark
file:///PATH_TO_DATA/config.ttl#0.056127920957769306 http://purl.org/dc/terms/created 2019-02-14T06:14:55.926Z
file:///PATH_TO_DATA/config.ttl#0.056127920957769306 http://purl.org/dc/terms/title Open Artificial Pancreas
file:///PATH_TO_DATA/config.ttl#0.056127920957769306 http://www.w3.org/2002/01/bookmark#recalls https://youtu.be/p76hGxv3-HE
file:///PATH_TO_DATA/config.ttl#0.056127920957769306 http://xmlns.com/foaf/0.1/maker file:///profile/card#me
```
Which you will recall are the values of the bookmark from the example in the [previous post](../using-rdf-data-1).

```
@prefix terms: <http://purl.org/dc/terms/>.
@prefix bookm: <http://www.w3.org/2002/01/bookmark#>.
@prefix n0: <http://xmlns.com/foaf/0.1/>.
@prefix c: </profile/card#>.
@prefix you: <https://youtu.be/>.

<> terms:references <#0.056127920957769306> .
        
<#0.056127920957769306>
    a bookm:Bookmark;
    terms:created "2019-02-14T06:14:55.926Z"^^XML:dateTime;
    terms:title "Open Artificial Pancreas";
    bookm:recalls you:p76hGxv3-HE;
    n0:maker c:me.
```

Finally, we have our setup that will allow us to move towards some of the questions asked in the first post.

In the next post we will look at how `graphql` might help solve some of the questions posed above.
