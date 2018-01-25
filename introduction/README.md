# NiFi tutorial

This tutorial is intended to introduce some of the basic semantics of working with Apache NiFi. It is assumed that you are somewhat
comfortable with [Docker](https://docker.com), and currently this tutorial also assumes that you are using some sort of Unix/Linux, and are comfortable with the command line.

The tutorial will first walk you through:
 - how to get a local instance of NiFi running;
 - how to basically interact with the NiFi interface;
 - writing a system that's a bit more complex than 'Hello World';
 - how to clean up and shut down the local instance;

## Running NiFi with Docker

We will start by pulling and executing an official [Apache NiFi](https://hub.docker.com/r/apache/nifi/) Docker image. Before following the instructions below, first change to a directory that you will be comfortable sharing with the executing Docker image, such as a local documents or project folder.

```
docker login
docker pull apache/nifi:1.5.0
docker run --name nifi -p 8080:8080 -d -v $(pwd):/mnt apache/nifi:1.5.0
docker logs -f nifi
```

This will start the image running, with the current directory shared with the `/mnt` directory inside the instance. The image is quite large, so be patient with the `docker pull`, and it takes a fair amount of time for NiFi to be available for work. When you seee a message like the following in the log, it's ready to work.

```
2018-01-23 08:58:12,146 INFO [main] org.apache.nifi.web.server.JettyServer
    NiFi has started. The UI is available at the following URLs:
```

Once you have seen this, you can connect to <http://localhost:8080/nifi> to access the user interface in a browser. Most modern browsers are supported.


## The User Interface
(stuff in here about what you can see)

## Introduction to FlowFiles
One of the [key concepts in NiFi](https://nifi.apache.org/docs.html) that you need to understand is the _FlowFile_. The official documentation says:

> A FlowFile represents each object moving through the system and for each one, NiFi keeps track of a map of key/value pair attribute strings and its associated content of zero or more bytes.

This is entirely accurate, but not necessarily particularly useful. A more useful way of thinking about a _FlowFile_ is as a virtual envelope. Each _FlowFile_ envelope _may_ contain some data _Content_, and has some information (_Attributes_) written on the outside as key/value pairs.

All NiFi workflows entail creating FlowFiles and then manipulating those FlowFiles as they move through the process. The attributes written on the outside of the "envelope" can be updated, added to, or deleted, and the content can be examined and manipulated more or less arbitrarily. FlowFiles can be split apart and merged together, the content can be used to update the attributes, and attributes can become content. As long as you hold onto the idea of an envelope with data inside it, you shouldn't have too much difficulty.

Another key concept is that of a _Processor_. Workflows are built by joining processors together. Each type of processor (and there are hundreds), can either consume FlowFiles and pass them along, generate new FlowFiles, or take FlowFiles and generate several new ones from them.

- Drag a _Processor_ onto the canvas, then use the _Filter_ to find the `GenerateFlowFile` processor, highlight it and select _Add_.

- Right-click on the new _Processor_ and choose _View Usage_. This should show the documentation for the processor. All processors, and many configurable items, have on line documentation which can help with using them. In this case, this processor generates new FlowFiles which optionally contain dummy data.

- Observe that the _Processor_ has a yellow warning symbol on it - if you hover the mouse over it, you will get a report of why there's a problem. In this case, the processor is not connected to anything: there is nowhere for the newly generated FlowFiles to go.

- Drag a new _Processor_ onto the canvas, and add a `LogAttribute` processor.

- Click on the `GenerateFlowFile` processor to the `LogAttribute` processor to connect them, and select _Add_ on the dialog that pops up. You should see that the `GenerateFlowFile` is no longer showing a warning - it's marked with a red square instead now - but the `LogAttribute` processor still has a problem.

- Right-click on the `LogAttribute` processor, and choose _Configure_, then the _Settings_ tab. We need to terminate the stream of FlowFiles, ending the flow at this final processor in the stream, so select to "automatically terminate" the `success` relationship. You will see later that some processors have multiple output connections ("relationships"), which allows routing of different FlowFiles in different directions. Select _Apply_ to make the change.

- Right-click on the `LogAttribute` processor again, choose _Configure_, then the _Properties_ tab. Set "Attributes To Log" to `filename` (careful, this is case sensitive), then select _Apply_.

- Right-click on the `GenerateFlowFile` processor again, and chose the _Scheduling_ tab. Set the run schedule to 5 seconds. Some processors (particularly those that receive FlowFiles) only 'wake up' when they have work to do. A processor that generates FlowFiles, like this one, will loop as quickly as the computer will allow unless we tell it otherwise! _Apply_ the change, and you should observe that there are no more warnings visible.

- Click somewhere on the canvas so that no processors are selected, then press _Start_ on the _Operate_ control set on the left of the window. The red squares should now change to green, indicating that the processors are running. Individual processors can be stopped and started as well - experiment to see what happens. You may need to click on the canvas and select _Refresh_ to see effects - by default the information on a running workflow only updates every few seconds (minutes in some cases).

- Stop the flow by making sure no processors are selected and pressing _Stop_ on the _Operate_ control set, then look at what was appearing in the log file. You should see entries that resemble:

```
--------------------------------------------------
2018-01-23 14:54:00,453 INFO [Timer-Driven Process Thread-1] o.a.n.processors.standard.LogAttribute LogAttribute[id=237d29cd-0161-1000-bb2a-34170cc4d3ab]
 logging for FlowFile StandardFlowFileRecord[
 uuid=9ecb1c13-ec4b-42e4-844a-0e32ce7046a8, claim=,
 offset=0, name=23601570659777, size=0]
--------------------------------------------------
Standard FlowFile Attributes
Key: 'entryDate'
	Value: 'Tue Jan 23 14:54:00 UTC 2018'
Key: 'lineageStartDate'
	Value: 'Tue Jan 23 14:54:00 UTC 2018'
Key: 'fileSize'
	Value: '0'
FlowFile Attribute Map Content
Key: 'filename'
	Value: '23601570659777'
--------------------------------------------------
```

- Note that the specified attribute `filename` is shown under "FlowFile Attribute Map Content", and other standard attributes are shown above. The `LogAttribute` processor is a very useful debugging tool, but you will discover others in time. The `filename` of a FlowFile is internally generated, and does not necessarily have anything to do with a "file" that is being processed: it is purely an identifier for the FlowFile. It can be updated if you need, but don't worry too much about the name.

- Right click on the `LogAttribute` processor again, and select _View data provenance_. You will see a list of recent FlowFiles that have entered or left the processor. Selecting the small "information" symbol at the start of each line will pop up a dialog that lets you examine the details, attributes and contents of the FlowFile. Explore this to see what you can find!

- Click on the canvas, then drag to select both processors, then use the backspace or delete key to remove them. The canvas should be empty before going on.


## Finding Satisfaction - a serious workflow.
The Rolling Stones famously could not find any [satisfaction](https://youtu.be/byn7ZUjz5DY), but we're going to dig through some books by Charles Dickens to see if it is there. In this workflow we are going to read a text file that has a list of URLs in it (references to books at [Project Gutenberg](http://www.gutenberg.org)), fetch each of the documents referred to, and then find the number of times the word "satisfaction" appears in each one. We'll write the results out to text files, but in a more realistic system we'd probably put them in a database or put them on a message queue for further action.

The workflow template is along side this document, and could be imported, however you will learn more by building it up manually. There are quite a few processors that will be added and configured for this workflow, which should introduce some basic patterns of use that you will find again and again.

- Drag a `GetFile` processor onto the (empty) canvas. This will be the start of our workflow - this processor periodically looks in a directory for files matching some criterion, then creates a FlowFile containing the data in that file.

- Right click on the `GetFile` processor to configure it, and choose the _Properties_ tab. Set **Input Directory** to `/tmp`, set **File Filter** to `urls.txt`, and set **Polling Interval** to `2 sec`. Finally, also set **Recurse Subdirectories** to `false`. Finish by selecting _Apply_ to save the changes. This configures the processor to look every 2 seconds for a file `/tmp/urls.txt`, and if it exists to create a new FlowFile with the contents. This processor will also (try to) delete the file after it has read it, unless configured otherwise.

- Drag a `SplitText` processor onto the canvas, and right click to configure it. On the _Properties_ tab set **Line Split Count** to 1 and select _Apply_ to save the changes. This processor will break the incoming FlowFile into multiple FlowFiles with 1 text line in each.

- Drag from the `GetFile` to the `SplitText` to connect them. By default the name of the connection is `success`, which is the name of the output Relationship of the `GetFile` processor. This can be updated if desired, but mostly there's little benefit.

- Drag an `ExtractText` processor onto the canvas, then right-click to configure its _Properties_. This time we won't update any of the existing attributes, but instead add a new attribute. On the upper right of the _Properties_ tab is a plus (+) button which adds a new property. Select this and enter `document_url`. Press _Ok_ to accept that name, and then add `^.*$` as the value of the property. This processor is used to examine the contents of the incoming FlowFile, and then create new FlowFile attributes for any data that matches a particular regular expression. We have just defined a regular expression that matches the whole line of text, and said that this line of text will become a new attribute called `document_url`. More or less... in fact the attribute will be called `document_url.0`, `document_url.1`, `document_url.2` etc, for each piece of data that matched the regular expression. Since our regular expression matches a whole line, and the incoming FlowFile will contain a single line, we will wind up with just `document_url.0` set to one of the lines from the `/tmp/urls.txt` file.

- Connect the `SplitText` processor to the `ExtractText` processor - this time a dialog will pop up asking which Relationship you want to make the connection for. The `SplitText` processor is an example of one which can send different information out of different connections. There are three: `failure`, `original` and `splits`. The `splits` connection will have the new FlowFiles created by splitting apart the incoming FlowFile, so choose that one and select _Add_ to save the change.

- The `SplitText` processor still has a warning. If you hover the mouse pointer over the exclamation mark you can see the reason for the warning - there are some connections that are not terminated or connected. Right-click the processor to configure it and choose the _Settings_ tab. Here you can auto-terminate the connections we are not interested in: select `failure` and `original`, and choose _Apply_ to save the setting. The warning should now go away.

- Drag an `InvokeHTTP` processor onto the canvas, and configure it. Under _Properties_, set **HTTP Method** to `GET`, and the **Remote URL** to `${document_url.0}`. This is an example of the [NiFi Expression Language](https://nifi.apache.org/docs/nifi-docs/html/expression-language-guide.html) - rather than setting the property to a fixed value, we are setting it to the value of a property `document_url.0`. This is a common kind of pattern: we use a previous processor (`ExtractText`) to set an attribute on the FlowFile in order to be able to use it in the configuration of later processors. Choose _Apply_ to save the changes.

- Connect the `ExtractText` processor to the `InvokeHTTP` processor, choosing the `matched` relationship, then configure the `ExtractText` processor to auto-terminate the `unmatched` relationship. This is another common pattern - making a connection, then going back to the source of the FlowFile to auto-terminate the unneeded relationships.

- We now want to add a step which just updates the attributes of the FlowFile being processed. Fortunately there is a processor just for this, so drag an `UpdateAttribute` processor onto the canvas, and go to its *Properties*. The name of this processor is a bit misleading, because it can update, delete and add properties. In this case use the plus (+) button to add a new property `filename`, with a value `${document_url.0:substringAfterLast('/')}`. This is an example of a more complex use of the expression language, where we are taking the URL and chopping out all but the last part. For example <http://www.gutenberg.org/files/766/766-0.txt> gives us `766-0.txt`. If you remember the first exercise, you would recall that `filename` is an auto-generated value created by NiFi on all FlowFiles. We are overwriting it with a desired value that we will use later on.

- Connect the `InvokeHttp` processor to the `UpdateAttribute` processor for the `Response` relationship, then configure the `InvokeHttp` processor to auto-terminate all other relationships. Notice that the *Settings* tab shows some information about each of the relationships. After you _Apply_ the change, the warning on the `InvokeHttp` processor should go away.

- At this stage, the FlowFile that comes out of the `UpdateAttribute` processor should contain the data fetched from the URL, and a bunch of attributes giving information about the FlowFile, the content, and the results of the previous steps. Attributes accumulate as the FlowFile moves through the process, unless you use an `UpdateAttribute` processor to explicitly remove them.

- We're going to use a snippet of code - a script - to process the contents of the FlowFile now. Each of the text documents at Project Gutenberg is several hundred kilobytes long, so it would be useful to have a small program that just digs around in that text to extract what we want. Drag an `ExecuteScript` processor onto the canvas, and connect the `UpdateAttribute` processor to it.

- Configure the `ExecuteScript` processor by changing the **Script Engine** property to `Groovy`, and the **Script Body** property to the Groovy program below (you probably should copy this and paste it into the Processor):

```
import org.apache.commons.io.IOUtils
import java.nio.charset.*

def flowFile = session.get()
if (!flowFile) return

flowFile = session.write(flowFile, {inputStream, outputStream ->
  def wordCount = [:]

  def textBody = IOUtils.toString(inputStream, StandardCharsets.UTF_8)
  def words = textBody.split(/(!|\?|-|\.|\"|:|;|,|\s)+/)*.toLowerCase()

  words.each { word ->
    def currentWordCount = wordCount.get(word)
    if(!currentWordCount) {
      wordCount.put(word, 1)
    }
    else {
      wordCount.put(word, currentWordCount + 1)
    }
  }

  def outputMapString = wordCount.inject("", {k,v -> k += "${v.key}: ${v.value}\n"})

  outputStream.write(outputMapString.getBytes(StandardCharsets.UTF_8))
} as StreamCallback)

session.transfer(flowFile, REL_SUCCESS)
```

- This script does more than we need, but was one that we already had lying around. It's a small [Groovy](http://www.groovy-lang.org) program that reads the contents of the incoming FlowFile, and counts the occurence of every word in it. The contents of the FlowFile are then replaced with a list of every word and the number of occurences in the form `word: number`, e.g. `"satisfaction: 16"`. The `ExecuteScript` processor supports a few different programming languages, and the script can be stored as a file external to the processor as well. While it is fairly straight forward for a Java programmer to create new Processors for specific purposes, this is often overkill for simple operations that can be performed with a simple script. Another common pattern is to do what we did here: use a pre-existing script that is pretty close to what we need, rather than writing a new one.

- The FlowFile content has been replaced with our word frequency list, which is great, but we don't want _every_ word, just the word "satisfaction". It would be nice if there was a Processor that could filter this list for matching lines and send the results somewhere. Fortunately the `RouteText` processor can do just that! Drag a new `RouteText` processor onto the canvas, and connect the `success` relationship from the `ExecuteScript` processor to it. Don't forget to go back to the `ExecuteScript` processor to auto-terminate unneeded relationships.

- Configure the properties of the `RouteText` processor to set `Matching Strategy` to `Starts With`, then add a new property `satisfaction` with the value `satisfaction:`. This is instructing the processor to create a FlowFile containing every line starting with `satisfaction:` and send it down a new relationship called `satisfaction` (we could of called it `matched` as an alternative for clarity).

- This processor has a pretty complex behaviour, and it's a good example of one where the documentation is not particularly informative. Right click on the processor and select _View Usage_ to explore the documentation.

- The final thing we want to do is preserve our FlowFiles, which now contain the count of the word "satisfaction" for a specific document fetched from the URL. Drag a `PutFile` processor onto the canvas, and connect the `RouteText` processor to it. You will see that there is a relationship called `satisfaction`, so choose that one. Add the connection, then configure `RouteText` to auto-terminate the unneeded relationships.

- Configure the properties of the `PutFile` processor to set **Directory** to `/tmp/results` and **Conflict Resolution Strategy** to `replace`, then _Apply_ the change. Why do you think we're not specifying a file name? Remember that each FlowFile always has a `filename` property, and that we used the `UpdateAttribute` processor to set it to a value derived from the URL. This is the name of the file that the `PutFile` processor will try to write.

- Because this is the last processor in the workflow, we need to auto-terminate all of it's relationships, so right-click to go into it's *Settings* tab and do that.

- There should be no warnings on any processors at this stage, and we are ready to run the workflow. Click somewhere on the canvas to make sure that no processors are selected, then use _Start_ button on the _Operate_ control panel to start all the processors running.

- If you right-click on the canvas and choose _Refresh_, you will see that the workflow seems to be doing nothing! This should not be a surprise - the `GetFile` will be waking up every so often to see if `/tmp/urls.txt` exists, then doing nothing because it does not yet exist. We need to do something about that. You will recall when we started the NiFi Docker container, we shared the current directory with the `/mnt` folder inside the container. We can thus use this shared location to edit a file locally and copy it into `/tmp` inside the container.

- Open a new terminal window in the current directory, and create a file `urls.txt` there with the following content:

```
http://www.gutenberg.org/files/24022/24022-0.txt
http://www.gutenberg.org/files/700/700-0.txt
http://www.gutenberg.org/ebooks/1023.txt.utf-8
http://www.gutenberg.org/files/766/766-0.txt
http://www.gutenberg.org/ebooks/730.txt.utf-8
http://www.gutenberg.org/files/1400/1400-0.txt
```

 - next we will connect to the running NiFi instance to run commands inside it:

 ```
 docker exec -it nifi /bin/bash
 ```

- You should see the prompt change to show that you are now interacting with the command line inside the container:

 ```
 $ docker exec -it nifi /bin/bash
nifi@0dc61f8bd6cb:/opt/nifi/nifi-1.5.0$
 ```

 - list the contents of the share directory to verify our URLs file is there:

 ```
 nifi@0dc61f8bd6cb:/opt/nifi/nifi-1.5.0$ ls -al /mnt
 total 8
 drwxr-xr-x  3 nifi nifi   96 Jan 24 12:14 .
 drwxr-xr-x 60 root root 4096 Jan 25 08:46 ..
 -rw-r--r--  1 nifi nifi  279 Jan 24 12:08 urls.txt
 ```

 - our URLs file should be available, so let's copy it to `/tmp/urls.txt` so that the NiFi workflow will find it:

 ```
 cp /mnt/urls.txt /tmp/urls.txt
 ```

- go back to the NiFi console, and refresh the display. You should see that all the processors have now indicated they have done something recently, and in particular you should see that the `PutFile` processor has processed 6 FlowFiles. You should also notice that the `InvokeHTTP` processor has output about 7Mb of data, while the `PutFile` processor has only written out a few hundred bytes. See if you can observe how the raw data has been filtered down as the workflow has proceeded.

- go back to the command line console in the Docker instance, and look at the contents of `/tmp/results`:

```
nifi@0dc61f8bd6cb:/opt/nifi/nifi-1.5.0$ ls -al /tmp/results
total 32
drwxr-xr-x 2 nifi nifi 4096 Jan 25 13:06 .
drwxrwxrwt 7 root root 4096 Jan 25 13:06 ..
-rw-r--r-- 1 nifi nifi   17 Jan 25 13:06 1023.txt.utf-8
-rw-r--r-- 1 nifi nifi   17 Jan 25 13:06 1400-0.txt
-rw-r--r-- 1 nifi nifi   16 Jan 25 13:06 24022-0.txt
-rw-r--r-- 1 nifi nifi   17 Jan 25 13:06 700-0.txt
-rw-r--r-- 1 nifi nifi   17 Jan 25 13:06 730.txt.utf-8
-rw-r--r-- 1 nifi nifi   17 Jan 25 13:06 766-0.txt
```

- there should be six files there. Let's look at them to see if we found any satisfaction!

```
nifi@0dc61f8bd6cb:/opt/nifi/nifi-1.5.0$ cat /tmp/results/*
satisfaction: 35
satisfaction: 15
satisfaction: 1
satisfaction: 21
satisfaction: 16
satisfaction: 53
```

- observe too that our input file is no longer present:
```
nifi@0dc61f8bd6cb:/opt/nifi/nifi-1.5.0$ ls -al /tmp/urls.txt
ls: cannot access '/tmp/urls.txt': No such file or directory
```

- type `exit` to disconnect from the running Docker instance, and go back to the NiFi console. Try right-clicking on various processors to view their *Data Provenance*. See if you can observe how attributes are accumulated on the FlowFile as it goes through the process, and how the contents of the FlowFile can change as it progresses.

### Extension exercise
As an extension exercise, think about how we could speed processing up. The obvious thing would be to fetch all the documents from Project Gutenberg in parallel, rather than sequentially. The question of whether or not actions are happening in parallel is slightly complicated - in general terms each new FlowFile that our `GetFile` creates to start the process will see parallel, independent, instances of the workflow executing, one for each FlowFile,  unless we configure things otherwise. Within the workflow, by default, the processors will process each FlowFile sequentially. Remember that we started with one FlowFile from the `GetFile`, but split it into 6 new FlowFiles with the `SplitText` processor.

- Right-click the `InvokeHTTP` processor and **Stop** it, then right-click again to configure it. You cannot change the configuration of a running processor. Select the _Scheduling_ tab, and set **Concurrent Tasks** to 4. _Apply_ the change, then right-click the processor to **Start** it again.

- Do the same thing with the `UpdateAttribute`, `ExecuteScript`, `RouteText` and `PutFile` processors to do the same.

- Get back into the Docker image command line, and copy our URLs file back into `/tmp/urls.txt` again.

- Refresh the view in the NiFi console - you should observe that the workflow now has executed more quickly.

## Cleaning up

We can stop and discard our image now. Use Control-C to stop tailing the log, and then do:

```
docker stop nifi
docker rm -f nifi
```
