```bash
Q:

Design a Poll widget.

Here is one from twitter.

Maybe you can talk about:

data structure
expiration
data refresh policy
cache / bottleneck
something fun like animations .etc
a11y

```


It is a vague problem, we should narrow down the scope firstly and clarify the requirement and goal.

So, what is the Poll Widget?
----------------------------

The poll widget is a small website component for data collection. The user can create multiple questions and get a link for sharing. People who visit the link will see the options and vote for their choice. After they submit the vote, the polling result will be displayed.

## Requirements Gathering and Clarification
----------------------------------------

Functional Requirement:

1.  the creator can create the polling and get a sharing link (id)
2.  the creator can click the add button and enter text to create question options
3.  people who visit through the share link can vote and see the results
4.  the polling expires after a certain period
5.  how much style customization it supports

Non-functional Requirement:

1.  auto synchronizing data
2.  multi-device supported
3.  internationalization
4.  browser compatibility
5.  accessibility

## Capability and Constraints
--------------------------

1.  Once the polling is created, it cannot be modified
2.  The maximum number of options is 10
3.  The maximum text length of each option is 100 characters

## High-Level Design
-----------------

1.  UI when users are creating polling
2.  UI when people haven't voted yet
3.  UI when people have voted

**Data Entities and Relationships**

data model

-   PollDTO: id, title, options:option[], creatTime, expiration, votes:{id, votes}
-   PotionDTO: id, description

state

-   status: creating | voting | voted
-   options: PotionDTO[]
-   title
-   expiration

relationship

-   creating -> title + options + expiration -> get share link from server
-   visit from link -> server data -> render -> (vote) -> result 

## API Design
----------

id: null -> for creating mode / id -> retrieve data from server

token: to identify whether the user has voted

retrieveData(pollID, token) -> PollDTO

onCreatePolling(title, options, title, expiration, token) -> share link

onVote(pollID, optionsID, token) -> boolean

## Deep Dive
---------

### Necessary Animations

-   reflecting the user interactions
-   reflecting the updates of the result

### Realtime Result Display

*HTTP short-polling*

-   polling the server at every time interval.
-   simple to use
-   open and close connections frequently
-   consume network bandwidth

*HTTP long-polling*

-   open an HTTP connection and hold on until the update is sent from the server and received by the client. Then a new request is immediately opened to wait for the next update
-   lost connection on mobile networks
-   easy load balancing
-   simple to use
-   create and close lots of connections if the updates are very frequent

*Server-Sent Event*

-   instead of opening another connection after receiving the update, the server-sent event keeps a long-lived connection for receiving the stream of updates
-   doesn't need to create and close connections frequently
-   save network bandwidth
-   auto re-connect mechanism
-   need http/2 compatibility, otherwise, there is a maximum connections limitation, which causes issues when multiple tabs are opened
-   uni-directional communication, it only allows sending data from server to client. Extra HTTP requests are required for sending data to the server.

*WebSocket*

-   WebSocket establishes full-duplex communication over a single connection. It allows the client and the server to send messages to each other simultaneously
-   it is different from the HTTP request. Security, authentication, caching, and load balancing, all of them need to be re-considered
-   It involves extra libraries to your code. You should implement heart-beat, reconnection, etc, on your own
-   maximum connections limitations

Server-Sent Event wins because we don't need full-duplex communication and we should support the mobile network with low latency and reliability.

https://aquil.io/articles/a-comparison-between-websockets-server-sent-events-and-polling

### Internationalization

-   i18n-next library
-   maintain translated language packages
-   take care of the time zone, date format, text direction, measurements 
-   separated language files or being bundled together with components

https://strapi.io/blog/how-major-frontend-libraries-handle-i18n

### Multi-device support​

-   using rem for some element size
-   using CSS media query and flex layout
-   increasing the clickable area of interactive elements
