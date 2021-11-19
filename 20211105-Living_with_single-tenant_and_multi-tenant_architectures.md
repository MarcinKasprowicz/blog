---

Living with single-tenant and multi-tenant architectures
Lessons learned, observations and reflections.
As an engineering team in Schibsted, the largest media group in Scandinavia, we build common solutions for a wide range of organizations. I'm part a of the Interact team which is responsible for delivering core solutions to around 70 different newsrooms. Our two main products are Comments (click here to see our widget) and Live (click here to see an article on which it is embedded on). Comments are used by users which would like to take a part in a discussion, whereas Live is used by editors to cover different kinds of events (breaking news, sports matches, etc.).
We took two different approaches when designing our systems: In Comments, newsrooms operate on shared infrastructure, whereas for Live, it is dedicated per tenant. I'm going to share lessons learned during 2 years of working with those products. I hope that you will learn something interesting which will help you make better decisions when building systems that will be used by many organizations.

---

Ok, so in Comments, resources are shared. What does it mean? It means that each organization utilizes the same storage and operates on the same servers. As simple as that. With such an approach, each of the comments created in our database will have the newsroom's id assigned to it. This is known as a multi-tenant approach.
# comments table
| id | message                     | newsroom             |
|----|-----------------------------|----------------------|
| 1  | I love this article!        | VG (Verdens Gang)    |
| 2  | I have a different opinion. | BT (Bergens Tidende) |

Single-tenant and multi-tenant architecture compared on diagrams.
I really like the urban analogy pictured below. We talk about the single-tenant approach when every family lives in their own house. They don't share elevators, electric installations, staircases, etc. with other families. This is the opposite of living in an apartment building.

Estate of single-family houses versus apartment building. Photo on the left by Breno Assis and on the right by Samuel Ryde, both on Unsplash.
Let's now cover a few topics which differentiate those two approaches. For each of them I will share some examples from our projects, how they affected our work and where the gotchas 🎣 are. At the end of the article I will try to think about how I would design those systems if I had a chance o do it again from scratch. Yes, I heard about the second-system effect but I'm too weak to resist 😂.
Data isolation
Separate storage means that data from different tenants won't interfere with each other. It also should protect us from unrestricted data access. I have been trying to find an endpoint in Comments that would allow me to do a small hack and access to data which I shouldn't be able to. And I found it.

```
curl --request POST \
    --url https://host/publications/e24/comments/3159602/votes \
    --header 'Authorization: <user-logged-in-e24-newsroom>'
```

What is this request doing? As a user logged in to the E24 newsroom I voted on a comment identified by the number3159602 . However, this comment is not assigned to E24 but to Bergens Tidende 😨. Comments from all newsrooms are living together in one big table. As a user in organization A, I was able to manipulate data in organization B. This is not a big deal in Comments because our API is not public and user accounts are shared across all newsrooms. Other systems might be much more concerned about data integrity. To mitigate such danger you would need to implement additional logic which restricts users from performing such actions. Developers make mistakes, there is always a chance that 🐛 will be introduced.
Application code in Live is tenant agnostic. It means that you will neither find any references to tenants in the code nor any conditional expressions with their identifiers. This reduces the likelihood of introducing bugs mentioned a few lines earlier.
Here is another example of code that isn't tenant agnostic and it is once again from the Comments project.

```
const votesCastTodayCount = user?.id
    ? await voteService.getUserVoteCountForToday(user.id, newsroomId)
    : 0;
```

The second argument of the method is newsroomId . You could find many more places where this additional argument is required. In Live you won't find any.
Tooling changes
During the last two years we spent quite some time migrating our projects from some platforms to others. Let's list a few of them
From Sumo Logic to Datadog as an observability solution.
From an in-house tool for provisioning to Terraform.
From one Redis provider to another.
From Heroku to Convox as a hosting solution.

What do you think, which system did we spend more time on? It was of course the single-tenant system. We have more manual work to do like copying configuration snippets, URLs, encrypting secrets, etc. This is simply because we had many more servers and databases.
Resilience
At the beginning of Comments, when the system was in its infancy, one of the newsrooms wanted to prepare something special for their readers. It was during NHL playoffs 🏒, and they wanted to increase user interactions. So they placed our widgets in multiple places on the index site (which we call the front page). As you can expect we started receiving an enormous amount of traffic on which we weren't prepared for. Usually, our widget is only placed on article pages. This sudden increase in the amount of traffic ended up killing our database 🔥. As the NHL games are being played at night when we are sleeping, comments across all Schibsted brands were down for a few hours. And what would be happening during a DDoS attack in a single-tenant system? Only one of the tenants would be affected whereas the rest of them would be operating as usual.
Since we are talking about taking services down, let's talk a little bit about recovery. How many of you have truly fully automated recovery procedures in place? Many things can go wrong when deploying a new version of a service: There can be a bug in the application code, a mistake in the database schema migration, or a mismatch in the DNS configuration. Quite often something unexpected happens, which you are not fully prepared for. In such a scenario, you need to take manual action to fix the problem. Would you like to revert the change on one database or on a number of tenant databases? In a system that is like Live you need to put some additional thought into disaster recovery and be prepared for it.
Adding a new tenant
In an ideal world adding a new organization in a multi-tenant (Comments) setup would only require extending the application configuration with an entry related to a new one. Maybe you would update your infrastructure by changing the autoscaling policy or you will just spawn a few new server instances. A single-tenant (Live) architecture would require one more step, and this step is related to infrastructure. You probably would need to add a new set of variables for each microservice, add new entries in your DNS configuration and update your monitoring dashboards. Sadly, the real world is usually more complex. Adding a new tenant in Comments doesn't really require much less work than in Live. Our application configuration is not centralized, it's scattered among microservices. We need to apply changes in each of our repositories. They're mainly related to preparing database schema migrations to add new records related to the new organization.
Let's imagine a situation that you are supporting a dozen tenants and at some point, your PM approaches you with a request of adding 50 new ones to your system. This scenario is exactly what happened this summer 😅. We needed to prepare Comments and Live to support 50 new newsrooms. For Live we supported 12 at that time. Spawning 50 new deployments (which in microservice architecture it means a lot of servers) would have cost us a lot 🤑! We needed to take a hybrid approach. We gathered those new newsrooms under one deployment. These were smaller newsrooms, local ones, from which we didn't expect big traffic. When going with a single-tenant architecture you need to ask yourself the question: Is it possible that at some point in time the interest in your product will be so high that everybody will want to use it?
I also feel obliged to say one more thing. Both products are built using the microservice architecture. Adding new tenants to Comments could take some time if you had to do this manually. Luckily, we have a simple CLI tool that does all the updates in our repositories. I strongly suggest you centralize the application configuration in the early stages. It can be a separate service with GUI or just a JSON file stored in S3.
Canary deployments (big features)
We are really happy with our current hosting solution which is Heroku, but during the summer we wanted to try something new. We quickly prepared Terraform configuration files and our CI/CD pipeline to support the new provider. It wouldn't be wise to move all of our infrastructure to a new hosting in one go. We decided to pick one of the newsrooms in Live for which we were observing low traffic and simply deployed it there. The risk wasn't big as deployments for Live are isolated.
Another interesting case where a single-tenant architecture can be helpful is when you are developing a big non-backwards compatible feature. We are currently working on such a thing in Live. During the release we will need to shut down the database and perform risky updates on it. We will start with a smaller tenant and gradually roll it out to others, fingers crossed 🤞.
Business intelligence
Our PM and clients often ask for all kinds of interesting data 💾 from our systems. Having one big relational database gives us the possibility to leverage the power of SQL and provide interesting insights in a blink of an eye. So this is how it's done for Comments. For Live we needed to prepare a separate service that is responsible for gathering data from all of our databases. We needed to invest some time in building it and it doesn't offer as much flexibility as a raw SQL solution.
Reverse proxy
To have one entry point for the system where each organization lives on a separate infrastructure per you will need to have a reverse proxy. That's one additional component in your infrastructure. For Live we use Fastly both as a CDN and a reverse proxy. Using its domain-specific language we can implement the following logic:

````
if (path.contains('VG')) {
    console.log('redirect to Verdens Gang');
} else if (path.contains('AB')) {
    console.log('redirect to Aftonbladet');
} else {
    throw Error({ code: 404 });
}
```

Reverse proxy pseudocode.
As a secondary solution we use Cloudfront together with a Lambda@Edge which is taking care of redirects.
Performance
One of the first thoughts when developers think about a single-tenant architecture is that it is a more performant solution. Tenants won't be sharing any resources between them so they will have more resources for themselves, right? And a thought came to me 💡. Would you prefer to have 20 shared instances or 2 instances per tenant, let's say when supporting ten of them? I think 20 shared instances would help you distribute potential traffic spikes and autoscaling for them wouldn't need to be aggressive. Response times should be the same for all and servers should be used much more efficiently. I would love to hear your opinions on that 👂. I know that I mentioned the NHL playoffs case but that was an extreme example and the Comments project was still in its early stages.
My recommendations
Go with a multi-tenant architecture but with separate namespaces in the databases. This will allow you to have a tenant-agnostic application code. You won't need to pass a tenant identifier around the codebase because you will have data isolation for free. It means less of conditional statements and no need for access checks to prevent cross-tenant actions. Much less ops work is also an advantage. If you are afraid that your relational database won't keep up I can assure you that relational databases are more powerful than you think (Postgres FTW 🚀, more about that in an upcoming article where I compared Redis, Dynamodb and Postgres). Focus more on optimizing your queries and models as well as on automatic reverts of different moving parts and a centralized configuration.