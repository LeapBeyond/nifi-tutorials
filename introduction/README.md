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

All NiFi workflows entail creating FlowFiles, then manipulating those FlowFiles as they move through the process. The attributes written on the outside of the "envelope" can be updated, added to, or deleted, and the content can be examined and manipulated more or less arbitrarily. FlowFiles can be split apart and merged together, the content can be used to update the attributes, and attributes can become content. As long as you hold onto the idea of an envelope with data inside it, you shouldn't have too much difficulty.

Another key concept is that of a _Processor_. Workflows are built by joining processors together. Each processor (and there are hundreds), can either consume a FlowFile and pass it along, generate new FlowFiles, or take a FlowFile and generate several new ones from it.

- Drag a _Processor_ onto the canvas, then use the _Filter_ to find the `GenerateFlowFile` processor, highlight it and select _Add_.

- Right-click on the new Processor and choose _View Usage_. This should show the documentation for the processor. All processors, and many configurable items, have on line documentation which can help with using them. In this case, this processor generates new FlowFiles which optionally contain dummy data.

- Observe that the Processor has a yellow warning symbol on it - if you hover the mouse over it, you will get a report of why there's a problem. In this case, the processor is not connected to anything: there is nowhere for the newly generated FlowFiles to go.

- Drag a new _Processor_ onto the canvas, and add a `LogAttribute` processor.

- Click on the `GenerateFlowFile` processor to the `LogAttribute` processor to connect them, and select _Add_ on the dialog that pops up. You should see that the `GenerateFlowFile` is no longer showing a warning - it's marked with a red square instead now - but the `LogAttribute` processor still has a problem.

- Right-click on the `LogAttribute` processor, and choose _Configure_, then the _Settings_ tab. We need to terminate the stream of FlowFiles, ending the flow at this final processor in the stream, so select to "automatically terminate" the `success` relationship. You will see later that some processors have multiple output connections ("relationships"), which allows routing of different FlowFiles in different directions. Select _Apply_ to make the change.

- Right-click on the `LogAttribute` processor again, choose _Configure_, then the _Properties_ tab. Set "Attributes To Log" to `filename` (careful, this is case sensitive), then select _Apply_.

- Right-click on the `GenerateFlowFile` processor again, and chose the _Scheduling_ tab. Set the run schedule to 5 seconds. Some processors (particularly those that receive FlowFiles) only 'wake up' when they have work to do. A processor that generates flow files, like this one, will loop as quickly as the computer will allow unless we tell it otherwise! _Apply_ the change, and you should observe that there are no more warnings visible.

- Click somewhere on the canvas so that no processors are selected, then press _Start_ on the _Operate_ control set on the left of the window. The red squares should now change to green, indicating that the processors are running. Individual processors can be stopped and started as well - experiment to see what happens. You may need to click on the canvas and select _Refresh_ to see effects - by default the information on a running workflow only updates every few seconds (minutes in some cases).

- Stop the flow, and look at what was appearing in the log file. You should see entries that resemble:

```
--------------------------------------------------
2018-01-23 14:54:00,453 INFO [Timer-Driven Process Thread-1] o.a.n.processors.standard.LogAttribute LogAttribute[id=237d29cd-0161-1000-bb2a-34170cc4d3ab]
 logging for flow file StandardFlowFileRecord[
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

- Note that the specified attribute `filename` is shown under "FlowFile Attribute Map Content", and other standard attributes are shown above. The `LogAttribute` processor is a very useful debugging tool, but you will discover others in time.

- Right click on the `LogAttribute` processor again, and select _View data provenance_. You will see a list of recent FlowFiles that have entered or left the processor. Selecting the small "information" symbol at the start of each line will pop up a dialog that lets you examine the details, attributes and contents of the flow file. Explore this to see what you can find!

- Click on the canvas, then drag to select both processors, then use the backspace or delete key to remove them. The canvas should be empty before going on.


## Finding Satisfaction - a serious workflow.

_TBC_

## Clean up

```
docker stop nifi
docker rm -f nifi
```

https://nifi.apache.org/docs/nifi-docs/components/org.apache.nifi/nifi-standard-nar/1.5.0/org.apache.nifi.processors.standard.ExtractText/index.html

document_url.0

data provenance!!!

http://www.gutenberg.org/files/24022/24022-0.txt
http://www.gutenberg.org/files/700/700-0.txt
http://www.gutenberg.org/ebooks/1023.txt.utf-8
http://www.gutenberg.org/files/766/766-0.txt
http://www.gutenberg.org/ebooks/730.txt.utf-8
http://www.gutenberg.org/files/1400/1400-0.txt



satisfaction

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
