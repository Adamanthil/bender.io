---
layout: post
title: Paying Off Our Technical Debt
---
All new businesses incur some amount of debt whether in the form of loans, investment notes, owners equity, or any number of other things. Many software companies also incur technical debt. [Technical Debt](http://en.wikipedia.org/wiki/Technical_debt) refers to employing software practices that provide expedited solutions at the expense of long-term development efficiency and maintainability. As with any kind of debt, the short-term benefits provided can sometimes be essential to delivering an actual product in time, given the [nature of software projects to run behind schedule](http://www.quora.com/Engineering-Management/Why-are-software-development-task-estimations-regularly-off-by-a-factor-of-2-3/answer/Michael-Wolfe?srid=24b). However, if the quality of code is sacrificed for speed in the short term, it must be paid back with interest later. Paying interest on technical debt amounts to extra work that would not have to be done if no corners were cut originally.

For us this interest rate was high.

More serious than any issues with quick solutions or corner-cutting, however, were dramatically shifted requirements from our original product design. [Simple Campus Housing](http://simplecampushousing.com) provides a web-based software package to colleges and universities for managing their on-campus housing and residence halls. When first prototyping our product for prospective universities, there was a strong aversion to using any software that was not hosted on campus servers behind their firewalls. In an effort to mitigate these concerns and support the wide array of possible IT infrastructures, we chose as ubiquitous and cross-platform a technology we could at the time: PHP.

#### Original Architecture
Our original architecture was a single PHP application per client using the [Lithium](http://li3.me) MVC framework and [Doctrine](http://www.doctrine-project.org) ORM. This allowed us not only to host the code on different operating systems and web servers, but also use whatever relational database technology each client had available. At one point we had production systems running on Linux, Apache, and PostgreSQL; Linux, Apache, and MySQL; Windows, IIS, and SQL Server; plus dev environments on OSX, Apache, and SQLite all using the same code base. This stack worked well for a while and was great to get us in the door, but we ran into maintainability and scalability problems quickly. Even from the start, we had to break out our roommate matching system into a separate web service, since we needed more control over the server resources and availability. As the complexity of the app rose, we quickly realized we needed complete control over all aspects of the environment to ensure the best user experience (and to support features we wanted to build).

Fortunately, the overall climate of IT in higher education has warmed significantly to the benefits of using cloud services since our first prototypes in 2009. We were able to move all code into our own cloud a couple years ago, which was a great first step. However, running an entirely separate server instance for each client was far from ideal.

We decided in June of last year that the time had come to begin a serious effort paying off our technical debt to really more forward. Our original architecture decisions were causing more and more issues with scaling, bugs, and increased development time. I've heard it said that "technical debt is often paid for by operations." But when you are a team of three, development _is_ operations. Something definitely had to be done.

#### New Architecture
Even though reading and particularly refactoring code is often much slower than writing new code, we wanted to avoid falling into the [old Netscape trap](http://www.joelonsoftware.com/articles/fog0000000069.html) of rewriting from scratch. Since our ideal new architecture was going to look significantly different from the monolithic PHP app we had built, however, much of it would have to be rewritten over time. This ideal architecture would be much more microservices-oriented, and allow us to chose the best programming tool for each task without affecting other parts of the application. Additionally, it would allow us to consolidate clients instead of requiring completely separate servers for each one. We also felt very strongly that we needed to support isomorphic code on the frontend. The real-time room selection feature we built relied a litany of complex business logic that had to be coded and debugged in the frontend javascript as well as the backend PHP. Given its complexity, maintaining and updating that system alone was proving to be a significant investment. Finally, we had to move away from an ORM. As convenient as it was in supporting disparate databases early on, the inherent inefficiencies were causing significant performance problems. Additionally, ORMs by their nature tightly couple code throughout an application, and we needed to break this coupling to improve maintainability.

We knew the business would not be served with new development out-of-commission for a significant period, so we decided to take a slow and incremental approach. We focused first on two key areas that would allow us to serve primary business objectives while moving us toward an updated technology stack. The first was decoupling our code from the ORM where it was least performant and most intertwined. In particular, this meant rewriting our "property system" which provides a way to generically address any piece of data in the system. This system is used by many other components from roommate matching to reports, and rewriting it would alleviate the worst performance problems and allow us to communicate with the database without relying on our PHP ORM. We built a new system using using direct PostgreSQL queries and a node.js backend that improved performance by about 2 orders of magnitude. This node backend would also provide the basis for our eventual service API that would route requests for all components in the future. The second area we focused on was authentication. We needed to support single-sign-on (SSO) for a number of clients, and rewriting our authentication system also removed the primary barrier we had to consolidating our servers. We replaced native PHP with [CAS Authentication](http://en.wikipedia.org/wiki/Central_Authentication_Service). This enabled us to use the same method whether users needed to be authenticated through our internal system or the single-sign-on server at their university. Combined, these changes allowed the business to reduce its growing server costs while delivering required feature updates and improved performance for clients.

![Architecture Transition](/images/architecture-transition.png)

The above diagram shows the current state of our system. A request is directed to the correct service by haproxy, which also adds a custom http header to identify the originating client. Much of our code still exists in the old php app, with authentication and a few other components stripped out. Over time we will rebuild portions of the php in our new system as we need to upgrade them and add functionality. [Jasig CAS](https://github.com/Jasig/cas) now provides the basis for our internal authentication, and communicates with a dedicated node.js service to determine client credentials. Some ajax requests are transmitted directly to the new node API over http. Other backend requests to the API can occur over [RabbitMQ](http://www.rabbitmq.com/) for increased speed. Client configuration settings and account credentials are now stored in a [Redis](http://redis.io/) database, while each client has an independent PostgreSQL database for main application data.

The primary transition to this architecture happened in February, and overall went quite well considering the scope of the changes. Since then we've added the RabbitMQ updates, fleshed out our SSO support, improved our frontend build process, and transitioned some smaller components. Most importantly, we have been able to continue making updates to the old application throughout this entire transition in service to our clients.

#### The Future

We are now working on new features, and one of the most exciting goals enabled by our architecture transition: Isomorphic Clojure\[Script\]. While this project is still in its early stages, we are very enthusiastic about what we've seen so far. Our current plan is to use [Om](https://github.com/swannodette/om) with Facebook's [React](http://facebook.github.io/react/) to build pages, rendering them on the backend using Clojure and Nashorn, and on the frontend with ClojureScript in the browser. I hope to post more about the details as we progress.