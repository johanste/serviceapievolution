# Versioning and compatibility of service APIs and client libraries

## Introduction

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

#### Examples of service API changes
The list of changes below is illustrative and not intended to be exhaustive.

Type of changes:

0. Allowed within the same API version.
1. Change that requires a new API version
2. Contract or semantically breaking change

|Change|Type|Notes|
|-|-|-|
|Addition of field in response|1|Deserialization may fail. Additionally, field may also get lost/not round-tripped in subsequent requests.
|Addition of optional field in request|1|Assuming that the new field does not change the interpretation of existing fields.
|Making optional field in request required|2|Requests from existing clients will fail unless they provide the value.
|Removal of field in response|2|
|Change unit of field from seconds to minutes|2|
|Change of range of numerical field in response from 32bits to 64bits|2|Existing clients may fail to parse returned values correctly. 
|Switch from "sync" (return 200/201 on initial request) to a "long running operation"|1|See [stepwise long running operations](https://github.com/microsoft/api-guidelines/blob/vNext/Guidelines.md#1322-post) in the Microsoft REST API Guidelines.
|Add support for conditional requests (etag/if-match)|1|
|Addition of `Retry-After` header|1|Clients are expected to honor the `Retry-After` header before making follow up requests for polling or redirects.
|Addition of rate limiting quota headers|0|See [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines/blob/vNext/Guidelines.md#1442-rate-limits-and-quotas) for more information.
|Addition of server-driven pagination support|1|Addition of `nextLink` in response. See the [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines/blob/vNext/Guidelines.md#981-server-driven-paging) for more information.
|Addition of new error codes in error condition responses|1|See the [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines/blob/vNext/Guidelines.md#7102-error-condition-responses) for more information.
|Addition of support for new verb (e.g. PATCH)|0|


##### Additional aspects to consider before allowing a service API change

- Incorrect (or imprecise) documentation of the service API forces users of the API to make explicit or implicit assumptions about the API. If a service always sent a property in a response and later stops doing so for the same input, the change is likely going to cause incidents regardless of if the attribute/property was marked as optional in the documentation.
- If a change exposes an issue in a previously released Microsoft official library, we can with a high degree of confidence assume that customers will have production workloads that are negatively affected by the change. This is true irrespective of if the issue is an actual bug in the client library.

## Issues with current guidance

### Issue 1 - Service API versioning scheme

From a customer's perspective, a date based versioning scheme does not carry any semantic information as to how large the change between two API versions is. I have anecdotal evidence that the fact a developer cannot easily determine if a change is large or trivial is cause for dissatisfaction.

An alternative would be to use a variation of semantic versioning rather than date based versioning. Minor version would mean additive only, major version for anything beyond additive changes.

> NOTE: This is primarily an issue where we have "contractual or semantical breaking changes" between api versions, since if the only changes we introduce between service api versions are additive (and service teams have fully internalized that those are the only valid type of changes), there is no need to signal larger changes between API versions.

### Issue 2 - Service API guidance (or interpretation thereof)

For service teams, the existing versioning guidance documents labels all kind of changes that require a new api version as "breaking". Unfortunately, this usage of the term "breaking" does not resonate with service developers understanding of "breaking" from other domains. The most common interpretation is semantic versioning, where "breaking" is generally associated with "major" version changes.

One unintended consequence of this guidance is that services either don't introduce new API versions because they believe their change is not breaking (it would be a minor version per semantic versioning rules). 

> For a recent example, see https://github.com/Azure/azure-rest-api-specs/pull/8585

Another unintended consequence is that since clients are shielded behind an API version, service teams do not distinguish between making *changes that require a new API version* and *contract or semantical breaking changes*. Overestimating the value of changes and underestimating the cost for customers to adopt *contract or semantical breaking changes* is a very common tendency among product teams - not just in Azure. The .NET Framework compatibility council has many years of experience of communicating this fact to partner teams.

> TODO: Insert specific examples or verbatims here

### Issue 3 - Client library specific issues with the earlier azure API versioning guidance

The previous official versioning guidelines state that each new API version corresponds to a new major version of a client library. This has several challenges:

- Many languages/runtimes are unable to load multiple versions of the same library in the same process.
- New major library versions cause great disruption in those ecosystems and should be avoided. One of the most common issues is the diamond reference problem where all libraries have to agree on the one and only version they reference.
- For language ecosystems where semantic versioning is used for library versioning, the bar for what a semantic major version is differs from the bar for "changes that require a service api version update". Many "changes that require a service api version update" are minor version changes according to semantic versioning. Strictly enforcing new major version of client library per API version would introduce unnescessary major version.
- An alternative to versioning packages is to create packages with the version number in the package name (e.g. instead of having `Azure.Storage` v17 followed by `Azure.Storage` v18, you have `Azure.StorageV17` v1 followed by `Azure.StorageV18` v1). However, this causes an explosion of the number of packages. This impacts how easy it is to find the correct package for a given service (the package manager's search engine generally boost the most commonly downloaded package, which means that new versions are effectively hidden from search).
It also prevents tooling from helping you upgrade versions since tools don't understand that the different packages can be treated as different versions of the same package. 

As an example of how many major versions of a given package we are currently releasing, [azure-mgmt-storage](https://pypi.org/project/azure-mgmt-storage/8.0.0/) for Python has had 5 major version changes since 2019.

## Proposed changes and additions

The proposed changes below are in addition to providing more tooling to both help review and detect changes as they are deployed. It is fully understood that guidance alone does not solve the problem.

### Proposal 1 - Updated SDK versioning proposal

In order to address [Issue 3](#-Issue-3---Client-library-specific-issues-with-the-earlier-azure-API-versioning-guidance) listed above, the updated Azure SDK client library proposal is to, instead of introducing a new major library version for a new API version, *add support for multiple service api versions in a single library*. By default, the latest service API version at the point of the release of the library is used, but a developer can opt in to earlier API versions.

The exact mechanism by which a developer picks which API version to use varies by language, but it generally involves passing in a specific Api Version to use when constructing a client objet.

Example from the [.NET Azure SDK design guidelines](https://azure.github.io/azure-sdk/dotnet_introduction.html#): 
```C#
public class ConfigurationClientOptions : ClientOptions {

    public ConfigurationClientOptions(ServiceVersion version = ServiceVersion.V2019_05_09) {
        if (version == default) 
            throw new ArgumentException($"The service version {version} is not supported by this library.");
        }
    }

    public enum ServiceVersion {
        V2019_05_09 = 1,
    }
}
```

It is worth pointing out that the tolerance for changes that may break a small subset of customers is higher between versions of a client library than for a service api since a developer/customer has to make an explicit change to their application in order to reference a new version of a library. This allows a customer to properly change their application on their own schedule before deploying it into production.

The proposal is not entirely without issues. The ability to support multiple API versions is dependent on services not regularly making large *contract or semantical breaking changes* since those types of changes would be hard to abstract away in a client library. However, this class of changes would be problematic in the previous guidance as well, since there is a strong correlation between changes that are difficult to abstract away in a client library that supports multiple API versions and changes that would cause friction in updating to a new version of a client library.

### Process and terminology changes

#### Update the terminology in guidance from "breaking" to "changes that require a new API version".

> TODO: I'd love to get proposals of better names, please...

Teams assume they understand what breaking means without reading our guidelines. And they often use the incorrect definition of the term.

#### Establish a clear approval process for larger changes.

Establishing a two-tiered process with both an escalation and approval paths for truly breaking changes helps team understand expectations. As part of reviewing the API changes, the Azure REST API review board can help determine if further approval is necessary.

(This assumes that teams actually go to the review board for api change proposals in the first place).

By requiring teams to present the business case and considered, but rejected alternatives for an impactful change, and get approval from Azure's Office of the CTO, it strongly encourages teams to do a more thorough investigation of less impactful  options.

#### Provide examples (good and bad) of previous service changes and provide feedback loop for customer incidents into the example document.

As part of the existing root cause analysis, we should include an analysis of if a specific change in service behavior was covered by the versioning guidance. Having a cookbook that includes patterns that can prevent future incidents as well as 
