### PGen: Evolution of an Application

## Personalized Passwords - the Idea

Building applications has had an interesting history and looking back at an evolution of a personal project over the course of several years can often provide us with a map of evolving ideas and technology that led us to make certain design decisions. This is often what happens in large tech companies (or, at least, I hope it does) where codebases are refactored and new technology is incorporated.

I remember having an idea of creating personalized passwords to enable more seamless login experiences but also aid users in password recollection. There were two main goals that the application was set to accomplish:

1. Enable seamless login experience
2. Make it easy to remember personalized passwords

**PGen** (short for Password Generator) was the app that I was going to build to accomplish these two goals. The user workflow would require users to pick a main topic and associated subtopic and provide password requirements.

![pgen_0](/assets/images/pgen/pgen_0.png)

Right before I even built the app I remember trying to figure out how to monetize it. This was a great exercise for me to understand what were the pitfalls in try to monetize an application. I don’t think the basic rules for understand B2C and B2B users has changed. There are tons of books that can help you understand the importance of scale for Consumer applications and building a truly good Enterprise product. I’ll write more about this towards the end of this blog.

This blog will talk about the evolution of building an application starting with web servers to SOAs and finally GenAI. I will share my learnings as I was building the final GenAI version of this application. There are a lot of resources out there but there were definitely some interesting challenges that I encountered.

# Version 1:  Web Servers

With the booming internet age, all of us became familiar with Apache web servers which served static and dynamic content for users. Then came the age of Service Oriented architectures for reusing services in enterprise land and more specifically Microservices for architecting individual applications. The goal was to build out modular applications with each module or service specifying a clean interface with which it would interact with other modules. These interfaces were usually REST APIs that defined the input/output contract between service modules. 

We are often influenced by what we see around us and what we are familiar with. Nothing could be more true than when I started to build the first version of this app. I had some experience working as a backend engineer in one of the teams at AWS. I quickly decided on putting together a quick client/server application using Apache Tomcat server and using Java to build out the application. I had done some PHP and javascript programming in my graduate school building out a website. However I knew that people were using AngularJS and I decided to learn it. The general architecture that I started out with looked something like this:

![pgen_1](/assets/images/pgen/pgen_1.png)

You could break down the system into the following categories:

1. UI
2. Backend
3. Database
4. Authentication
5. Security

# UI

For engaging enough users, you need to devise an application to require the smallest number of clicks or input. Initially, I had a complex workflow that required users to register with a password service and provide topics that they were interested in. I realized that this meant that for any new user, there would be a two step workflow for them to get a simple password. The overhead of registration would make the usability untenable. There was also a delay associated. Registration → Send topics interested to the server → Receive potential passwords. 2 RTTs at a minimum in my calculation. The UI was also a separate process or instance that needed to be spun up in addition to the backend server.

# Backend

For the backend I used Java to build out a server application. I was familiar with REST APIs which is what I used. This was probably the one decision that stuck through most of the remaining versions of the application except the last one. I love REST APIs. They are easy to build and you are able to define the input/output contract with the frontend. I was coding in Java in my day job so I used that which looking back might have been a more heavy weight language than I required. Initially I used the Spring framework but I built version 1 without it. 

# Database

Data is one of the most critical decision of any application. I don’t think I realized how much until I was well into building out the UI and backend above. Initially I wanted to get an E2E workflow going so I hardcoded what I wanted to see. For any application, you need to think about data - what are you requesting from the user, what are you storing and how are you going to protect it. For version 1, I decided that I was going to use scrape wikipedia pages to figure out what passwords for a given topic should be. For example, if a user is interested in “Tennis” then I should be scraping “Roger Federer”’s wikipedia page to identify other sub topic words that can be part of the password. So the password would be something like “tennis_roger” or “federer_goat”. The trouble was now maintaining a knowledge graph of User interested topics to wikipedia pages. I would have to store this mapping in a database and have sufficient sub topic pages to route to so that users get sufficiently different suggested passwords. There was also the question of performance if every topic selected by the user was going to need 1) scraping of a wikipedia page and 2) preprocessing to identify 2-3 sub topic words that are relevant to the main interested topic. For  2) which needed preprocessing I used the POS tagger website from the [Stanford NLTK tagger](https://www.nltk.org/_modules/nltk/tag/stanford.html). The tagger would preprocess the first 5-10 sentences of the website, and provide nouns and adjectives present on the page. The logic behind this was that we could combine nouns or nouns + adjectives to build out passwords based on specific wikipedia topic pages. 

# Authentication

For version 1, I decided on user registration and storing credentials in a database like SimpleDB which was one of the earliest Databases available in AWS. At this point I was still not clear about how I was going to launch this application and what endpoint would look like.

Though version 1 was a great learning process, there were obvious pitfalls that I did not see coming:

1. I spent time learning AngularJS. A lot of time and only because it was something that I saw online as potentially becoming the defacto language for UI development. Two teams and companies that I was working in were either rewriting their UI in Angular or were building products in Angular. Looking back this was not the best use of time. I should quickly hacked together HTML + JS pages especially when the application was not very complex.
2. In a similar vein as 1. I used Java to build out the backend for the application. Sometimes the tool can be more complex than required which was definitely the case here. 
3. I started out by writing down a basic block diagram of the BE architecture and building out the pieces needed from an execution point of view. I did initially have an idea about the product but a key thing that I missed was getting feedback from users. When I was deeper into the execution cycle I reached out to potential users and that led to a change in the workflow which was actually simpler than I had originally designed it.

## Version 2: Serverless AppEngine deployment

Learning from version 1 and reworking the user workflow, I decided on identifying a 2 click process to enable users to generate personalized passwords. With more experience I realized that PGen could be written as a serverless application. In version 1 I had decided on using AWS to deploy the frontend and backend, structure a security group and VPC and all the other service needed for exposing an externally visible endpoint and spinning out multiple EC2 instances. The PGen app now had the following workflow:

![pgen_2](/assets/images/pgen/pgen_2.png)

I made the following specific changes to account for this new direction in product and execution:

# UI

I skipped working on Angular realizing that I was not a FE engineer and never hoped to be one. I used plain old HTML and Jinja templates with a Flask App. I created a two click workflow. In the first page, the user would need to register or be authenticated as a previously registered user. In the second page, the user would provide information about topics that they were interested in building their passwords around and requirements just as numbers, special characters etc. to increase the strength of a password. With the click of a single button “Generate”, the third page would provide multiple password options.

# Backend

I identified a microweb server framework like Flask that could be written in a small amount of time using Python. At the time of version 2, I was using python heavily and had realized the benefits of fast iterative development. There are of course downsides but by this time I had decided that PGen would not be a large 100M user deployment. It was a step in determining product viability and anything that I wrote would eventually need to rewritten down the road. 

# Database

A key decision that I made when building out version 2 was to avoid storing data(user specific or otherwise). This was not counting the authentication tokens stored during user registration. Initially the database was set to contain a knowledge graph of topics and associated sub topics which would prevent having to go through the workflow of scraping wikipedia pages and processing them for the purpose of PGen. However maintaining a database is overhead that I was not ultimately able to justify. At a minimum, the only thing that the knowledge graph would have given me would be reduced latency. I decided to have every user request go through the entire workflow and handle performance optimization work as part of it. 

# Authentication

Registering the user was initially important as a way to tabulate number of requests made by each user. The goal was to provide a free tier usage with 5 free passwords per month. Enterprises could enable PGen as part of their login workflow and this would be a way to monetize the app. Initially I though of creating all the logic for user registration. This mean storing user ids and passwords or tokens of some sort. Any user information that you store has to have a layer of security. To prevent over engineering and learning from version 1, I decided to integrate the [Google oauth workflow into the flask application](https://realpython.com/flask-google-login/). There were definitely challenges here. I remember having trouble redirecting the user to the right URL and getting a token. It has been a while so maybe those challenges or issues have been solved and new ones are cropping up. Either way I know that this part of the application required understanding third party credentials and how to communicate with Google’s oauth service with the user’s permission. 

# Security

There is definitely HTTP web based vulnerabilities that we want to protect the user from such as cross side scripting. From an app POV I decided not to store user data or preferences. 

I deployed version 2 on AppEngine and you can check out the code [here](https://github.com/anj-s/pgen/tree/main/pgen_ver2). 

## Version 3: Generational AI Chrome Extension

The PGen app has been with me for a long time. I was very excited with the idea and wanted to build it out of seeing a genuine need for it. However in the process of building it I realized that it was a side project and monetizing would have had many challenges (see Monetizing section for challenges).  I decided to build my final version using LLM technology. 

I usually work on scaling LLMs and working on specific infra and optimization problems. I’ve used LLM APIs, run evaluation jobs and hit inference endpoints all with the express intention of ensuring the best performance or testing if something was working. It was never to identify the best prompt or integrating with an existing business application. I wanted to learn more about what the OSS world was building and approach this from a product perspective. It almost seems like the whole world is building off of the ChatGPT API and this seemed a good time to learn more. 

After building out version 2, I had often thought about making the user experience more seamless for PGen. One of the questions that kept cropping up was - Can I enable users to enable a PGen workflow right from the login page without explicit integration with website code? A more lightweight implementation I realized was a chrome extension that integrated into the user’s browsing experience. 

![pgen_3](/assets/images/pgen/pgen_3.png)

The other modification to the architecture was replacing wikipedia with LLMs as the knowledge source. LLMs are being used broadly in the following categories:

1. Knowledge base
2. Summarization or Analysis
3. Translation
4. Generation

With 1. there is always the chance that the LLM is going to contain stale data depending on when the LLM model was trained. However for the purposes of PGen, I was looking more for descriptive phrases about particular topics rather than factual data. ChatGPT prompt engineering for developers course on [DeepLearning.ai](http://DeepLearning.ai) was a great way for me to refresh and come at this problem from a developer POV.

# UI + Backend - Chrome Extension

I learnt about chrome extensions using the official [Google documentation](https://developer.chrome.com/docs/extensions/mv3/) and quickly spun up one of the examples. Using that as a template, I built out an initial version of PGen with hardcoded demo data. With chrome extensions, you have the ability to analyze DOMs and insert elements given certain conditions. This is part of the content scripts module that is part of the chrome extension. If you have an extension installed and you navigate to a website, you have the ability to process the DOM elements and identify if it is a login page. Once you have identified this, you can add a link or text into the web page. For PGen, on identifying that it was a login page of a given website (more on this specificity) I inserted a link to that would navigate to PGen landing page.  One of the challenges here is to identify a login page successfully since each webpage is different. It is possible to navigate to a web page and not see the PGen link. The other challenge is if we do want to explicitly parse every webpage for login page identification. We need to specify a list of webpages that we want content scripts to run in the context of.  The second entry point for PGen’s landing page is to click on the chrome extension similar to all other installed chrome extensions.

Chrome extensions can talk to backend servers that perform complex logic and store information in a database if needed. However I wanted the extension to be self contained and preferred the logic to reside in the extension itself. 

# Data

Having decided to use LLMs as a knowledge base, I spent some time identifying the best API. The constraint for me was that this was a side project purely for learning purposes and I did not want to budget any amount for paid LLM APIs. ChatGPT had an API subscription where calculating the total cost (using [OpenAI’s tokenizer](https://platform.openai.com/tokenizer)) for all prompts needed per password came to be:

```jsx
Input

4 prompts ⇒ ~ 80 tokens

1 followup prompt  ⇒ 30 tokens 

Total = 110 tokens ⇒ $0.165 * 10^-3

Output

26 tokens ⇒ $0.052 * 10^-3

Total amount per password ⇒ 0.217*10^-3
```

This means that even if I used the PGen app for a 1000 passwords, I would still be less than a $1 spent. However since PaLM APIs were free and my prompts were simple enough I went with it. We could just as easily add another LLM integration which is something I am planning to do.

For populating the initial topics that the user might be interested in followed by adjectives or other key words, querying the LLM was sufficient. The temperature was less than 1 to ensure that there were enough variety of responses for multiple users. The app was seeded by an initial list of main topics for users to choose from. Currently this is hardcoded but it would be easy enough to generate main topics based on relevant events and people in a given location.

![pgen_4](/assets/images/pgen/pgen_4.png)

I integrated Google Oauth2.0 signup using helpful blogs ([link1](https://medium.com/geekculture/googles-oauth2-authorization-with-chrome-extensions-2d50578fc64f) and [link2](https://github.com/firebase/quickstart-js/blob/master/auth/chromextension/credentials.js)). You need to add scopes to your manifest.json and use this API to retrieve the token. This is more if you are trying to access Google APIs or need a way to authenticate the used using his/her google identity. This is helpful if we are trying to track users in order to provide a free tier subscription. One thing to keep in mind is if we want the approval of the user to be silent or explicit. The silent approval for me did not work and I had to have the user explicitly navigate to the page where they approve the app logging in using Google credentials. 

# Security

The PGen app does not store passwords or any form of credentials. However in order to query LLMs we do ask for the user’s API key as a first step. We don’t store the key but we do use it for all LLM requests for the given user session. 
 

## Learnings

1. Prompting is an iterative process and currently there aren’t great tools for versioning, collaboration and evaluation. Playing with toy problems or trying out potential outputs on a playground interface is markedly different than way is needed for development. There is still some distance that we need to cover there. For output that is deterministic there can be a way to compare 1:1 but for subjective output or analysis, the comparison markers become more complex.
2. Parsing the output of LLMs is not a definitive process especially if you have settled for some randomness in the output. There were multiple output formats that I had to account for with my parser and am still convinced that that is probably the weakest part of the application. In this application, I am expecting a list of strings but imagine if the output were a complex JSON object with multiple entries. How do you deal with failed parsing in a real time application?
3. Using content scripts for parsing DOMs was amazing since the user would have a PGen entry point to click on. However every login page is designed differently so the surface area that I could target was much smaller than I expected. The chrome extension was still a much better entry point than having users navigate to an entirely independent standalone website.

## Improvements to the PGen App

There is always a lot that you can continue doing with any application. I think the sections above are a testament to how many times you can iterate to improve a product. Here are some short term ideas for continuing to improve PGen.

1. Add ChatGPT API support
2. Allow users to build out their own passwords but letting them enter a main topic.
3. Display multiple password options
4. Provide a thumbs up/down button to log immediate user feedback.

## App Monetization

PGen was initially just an application because I found it hard to come up with truly innovative personalized passwords. As I started flushing out the idea I realized that other folks must have the same issue. Talking to a few folks, my hypothesis was confirmed. I just wanted to build something cool and hence started working on it. I figured if I can make some money from it eventually that would be a good story as well. There was going to be a free tier subscription where each user was allowed 3-5 free passwords per month. This was going to be buttressed by an enterprise tier which was targeted towards companies that had thousands of new users per month. PGen was charge a nominal fee for enabling users accessing the login pages to use the PGen service. I did not have a set amount in mind but something to enable revenue after taking into account the cost of deploying this service on GCP.  As I built out the app, the monetization story was less appealing because of the tradeoff between effort and revenue. A few things I learnt that led to this decision and pivot for PGen to remain a fun side project:

1. There was no moat when it came to the idea itself. It was a fairly straightforward idea and one that could be easily implemented by an engineer in a few weeks. It would be hard to convince enterprise businesses to purchase a subscription when they could build it in house. 
2. To generate revenue from non enterprise end users, the app would need to scale massively. Though there was potential for usage, it was unlikely that users would need more than 3 passwords a month thereby falling within the free tier usage.
3. The idea itself came because of using multiple web applications which required some form of login. The products these days have been trending towards using Google or Meta’s authentication token. There will be a need for generating new passwords that don’t use this token but they are too few and far between to justify a subscription.

PGen was a great way for me to play with a lot different tools and technology. Here is a link to the [codebase](https://github.com/anj-s/pgen) with instructions on how you can download and play with the Chrome extension.