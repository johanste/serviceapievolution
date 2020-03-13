There are two primary, sometimes competing, priorities to Azure's service API versioning story:

1) **Ensuring that a correctly functioning application continues to function correctly.**

2) **Ensuring that service teams can provide new functionality over time.**

The primary mechanism by which this is accomplished is by allowing the client to specify an *api version* parameter for each request, where the service ensures that, given a specific api version, a client is never exposed to a change in behavior or structural contract. Only when the client explicitly opts in to the new behavior by providing an updated api version are they exposed to the changes.

Since a customer has no control over exactly when a service they are using gets updated, the rules for what changes are acceptable within a given API version are very restrictive. Almost no changes are allowed. For example, adding a read-only property to a response is not an approved change within a given API version. 

*Failure to adhere to the rules for what changes can occur within an API version risk introducing incidents and live site issues for our customers. This contributes to our customers perception of the *stability* of running workloads on the Azure platform.*

When service teams introduce new functionality behind a new API version, care must still be taken to not introduce unnescessary changes that cause friction for clients to adopt the new version. For example, the introduction of a new property in a response is a change that can easily be handled by a client that opts in to new functionality, whereas changing the structure of responses, removal of properties or change in semantics for the same input are all significantly harder to absorb.
While some developers, particularly early adopters of a service, may be more willing to accept these kind of changes, we know from years of experience from Windows and .NET that the majority of developers are not.

*The consequence of services making changes that are difficult for clients to absorb is that developers feel like Azure is difficult to write applications for due to breaking changes between API versions.*

This document uses the terminology *changes that require a new API version* to refer to anything that may cause an application to fail. It uses *additive features* to refer to situations where service API changes are purely additive and *contract or semantical breaking changes* for anything else.

In most cases, a client application that follows our best practices guidelines will continue to function with little to no change when upgrading to a new version that only had *additive changes*.

#### Examples of changes
The list of changes below is illustrative and not intended to be exhaustive.

Type of changes:

1. Change that requires a new API version
2. Contract or semantically breaking change

|Change|Type|Notes|
|-|-|-|
|Addition of field in response|1|Deserialization may fail. Additionally, field may also get lost/not round-tripped in subsequent requests.
|Addition of optional field in request|1|Assuming that the new field does not change the interpretation of existing fields.
|Making required field in request optional|1|
|Making optional field in request required|2|Existing clients have to provide the new value. 
|Removal of field in response|2|
|Change unit of field from seconds to minutes|2|

### Issues with the current service API guidance

For service teams, the existing versioning guidance documents labels all kind of changes that require a new api version as "breaking". Unfortunately, this usage of the term "breaking" does not resonate with service developers understanding of "breaking" from other domains. The most common interpretation is semantic versioning, where "breaking" is generally associated with "major" version changes.

One unintended consequence of this guidance is that services either don't introduce new API versions because they believe their change is not breaking (it would be a minor version per semantic versioning rules). 

> For a recent example, see https://github.com/Azure/azure-rest-api-specs/pull/8585

Another unintended consequence is that since clients are shielded behind an API version, service teams do not distinguish between making *changes that require a new API version* and *contract or semantical breaking changes*. Overestimating the value of changes and underestimating the cost for customers to adopt *contract or semantical breaking changes* is a very common tendency among product teams - not just in Azure. The .NET Framework compatibility council has many years of experience of communicating this fact to partner teams.

> TODO: Insert examples here

### Issues with the current service API versioning scheme

From a customer's perspective, a date based versioning scheme does not carry any semantic information as to how large the change between two API versions is. I have anecdotal evidence that the fact a developer cannot easily determine if a change is large or trivial is cause for dissatisfaction.

> NOTE: This is primarily an issue where we have "contractual or semantical breaking changes" between api versions, since if the only changes we introduce even between service api versions are additive, there is no need to signal larger changes between API versions.

### Client library specific issues with the earlier service API guidance

The previous official versioning guidelines state that each new API version corresponds to a new major version of a client library. This has several challenges:

- Many languages/runtimes are unable to load multiple versions of the same library in the same process.
- New major library versions cause great disruption in those ecosystems and should be avoided. One of the most common issues is the diamond reference problem where all libraries have to agree on the one and only version they reference.
- For language ecosystems where semantic versioning is used for library versioning, the bar for what a semantic major version is differs from the bar for "changes that require a service api version update". Many "changes that require a service api version update" are minor version changes according to semantic versioning.
- Publishing packages with the version number in the package name causes an explosion of the number of packages. This impacts how easy it is to find the correct package for a given service (the package manager's search engine generally boost the most commonly downloaded package, which means that new versions are effectively hidden from search). It also prevents tooling from helping you upgrade versions since tools don't understand that the different packages can be treated as different versions of the same package. 

For example, [azure-mgmt-storage](https://pypi.org/project/azure-mgmt-storage/8.0.0/) for Python has had 5 major version changes since 2019.

### Updated SDK versioning proposal

In order to address the issues listed above, the updated Azure SDK client library proposal is to, instead of introducing a new major library version for a new API version, add support for multiple service api versions in a single library. By default, the latest service API version at the point of the release of the library is used, but a developer can opt in to earlier API versions.

It is worth pointing out that the tolerance for changes that may break a small subset of customers is higher between versions of a client library than for a service api since a developer/customer has to make an explicit change to their application in order to reference a new version of a library. This allows a customer to properly change their application on their own schedule before deploying it into production.

The proposal is not entirely without issues. The ability to support multiple API versions is dependent on services not regularly making large *contract or semantical breaking changes* since those types of changes would be hard to abstract away in a client library. However, this class of changes would be problematic in the prvious proposal as well, since there is a strong correlation between changes that are difficult to abstract away in a client library that supports multiple API versions and changes that would cause friction in updating to a new version of a client library.

## Proposed changes to guidance and documentation

The proposed changes below are in addition to providing more tooling to both help review and detect changes as they are deployed. It is fully understood that guidance alone does not solve the problem.

#### Update the terminology in guidance from "breaking" to "changes that require a new API version".

> TODO: I'd love to get proposals of better names, please...

Teams assume they understand what breaking means without reading our guidelines. And they often use the incorrect definition of the term.

#### Establish a clear approval process for larger changes.

Establishing a two-tiered process with both an escalation and approval paths for truly breaking changes helps team understand expectations. As part of reviewing the API changes, the Azure REST API review board can help determine if further approval is necessary.

(This assumes that teams actually go to the review board for api change proposals in the first place).

By requiring teams to present the business case and considered, but rejected alternatives for an impactful change, and get approval from Azure's Office of the CTO, it strongly encourages teams to do a more thorough investigation of less impactful  options.

