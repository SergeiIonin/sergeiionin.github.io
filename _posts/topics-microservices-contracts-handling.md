## Challenges in the .proto Contracts in Microservices

There are quite a lot of reasons to choose gRPC as the protocol of [communication between microservices](https://hackernoon.com/creating-production-grade-microservices-with-go-and-grpc). It is also well known that microservice-architecture brings a tone of challenges,
including proper handling of the `.proto` files which are used for generation of client and server entities, and in particular - structures used for communication (classes in many programming languages).

The first question that may arise is where should the `.proto`-files reside? Each microservice may carry these files itselves, so that during the development we can alter `.proto` definitions in various ways, compile them and in the end we'll publish new artifact with compiled entities which other services can use. However, this approach has a couple of drawbacks especially when the number of microservices is large enough, additionally it introduces direct dependency between the microservices communicating with one another. And one of the biggest flaw of this approach is that we can't view the files used in the communication protocols (let's call them **contracts**) which is important if some service 
has multiple consumers and we need to be able to quickly find the data or service provider we need to integrate with.

Handling of such contracts used across the platform may consist of the following stages:

    1) Validate the contracts syntactically and semantically.
    2) Versioning of the contracts (which interleaves with semantical validation).
    3) Submit contracts through the pull request to the probably standalone contracts repository (where .proto files are hosted and probalby compiled) for human control.

The most intuitive approach of leveraging just a standalone contracts repository has a few drawbacks:

    1) There's no protection from breaking semantical validity of the contract, e.g. the contract with the following diff is syntactically correct:

"syntax = "proto3";
 - package users;
 + package users_new;
message GetUser {
  string id = 1
  string domain = 2
}

however, semantically such contract should be a different one. It's not a new version of the current contract since the package is updated.

    2) Provisioning of the semantical validity requires additional efforts.

    3) Lack of the navigation for a specific contract name (such as fetch all versions, fetch last version etc)

