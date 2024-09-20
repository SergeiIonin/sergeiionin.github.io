---
layout: post
title: "eithert-and-tapir"
date: 2024-09-20
categories: scala
---

## On Application of EitherT Monad Transformer from Cats to Tapir Server Endpoints in Scala
(In progress)

### Abstract
This article shows the concise and powerful technique to build server endpoints in `Tapir` library for Scala employing `EitherT` monad transformer from Cats Effect.  

### Endpoints and Server Endpoints in Tapir
[Tapir](https://tapir.softwaremill.com/en/latest/) is a modern Scala library for Scala which allows to describe HTTP typesafe endpoints using generic syntax, build [server endpoints](https://tapir.softwaremill.com/en/latest/server/logic.html) for higher-kinded type `F[_]` and interpret server endpoints for popular effect systems such as Cats Effect's `IO`, `ZIO` and `Akka` (NB and `Pekko`?). 

We could approach the endpoint definition in the following way (all DTOs are just some custom types):
```scala
  def root =
    endpoint
      .in("things")
      .errorOut(
        oneOf[HttpErrorDTO](
          oneOfVariant(StatusCode.BadRequest, jsonBody[BadRequestDTO]),
          oneOfVariant(StatusCode.InternalServerError, jsonBody[InternalServerErrorDTO])
        )
      )

  val createThingy =
    root
      .post
      .in(jsonBody[CreateThingyDTO])
      .out(jsonBody[CreateThingyResponseDTO])
      .name("CreateThingy")
      .description("Create new thingy")
``` 

Next we'll need to implement server logic for this endpoint:
```scala
  private val createThingySE: ServerEndpoint[Any, F] =
    createThingy.serverLogic(createThingy => {
      ??? // create(createThingy).value
    })
```
`serverLogic` from tapir's `Endpoint` accepts  `f: I => F[Either[E, O]]`, which in our case boils down to
`f: CreateThingyDTO => F[Either[HttpErrorDTO, CreateThingyResponseDTO]]`.
The result type `F[Either[HttpErrorDTO, CreateThingyResponseDTO]]` suggests to employ [`EitherT`](https://typelevel.org/cats/datatypes/eithert.html) in this case. 

### How to use EitherT in the server logic
There's quite a number of reasons to use `EitherT` or not as the primary effect type in the application. However, Cats ecosystem is flexible enough to allow us to flip to `EitherT` and back as necessary. Personally I find using of `EitherT` everywhere in the application as an overhead, but if I need to provide some HTTP-facing dependencies to server endpoints, [I'd rather wire them in `EitherT`](https://github.com/SergeiIonin/ContractsRegistry/blob/master/common/src/main/scala/client/CreateSchemaClient.scala):
```scala
final class CreateThingyServerEndpoints[F[_]: Async](
    clientCreate: CreateThingyClient[F],
    producer: EventsProducer[F, ThingyCreateRequestedKey, ThingyCreateRequested]
) extends CreateThingyEndpoints
```
then in the code that is actually used in server logic we can simply write

```scala
  private val createThingySE: ServerEndpoint[Any, F] =
    createThingy.serverLogic(createThingy => {
      create.value
    })

  private def create(
      createThingyDTO: CreateThingyDTO): EitherT[F, HttpErrorDTO, CreateThingyResponseDTO] =
    for
      response <- clientCreate.createThingy(createThingyDTO)
      ...
    yield CreateThingyResponseDTO(response.id)
```

It's obvious that for other dependencies we wouldn't pull HTTP error and result DTOs into the methods definitions (e.g. for producers of events). In this case we can either create an intermediate entity transforming the effect to `EitherT` or we can write the transformation right away in the function providing server logic:
```scala
  private val createThingySE: ServerEndpoint[Any, F] =
    createThingy.serverLogic(createThingy => {
      create.value
    })

  private def create(
      createThingyDTO: CreateThingyDTO): EitherT[F, HttpErrorDTO, CreateThingyResponseDTO] =
    for
      response <- clientCreate.createThingy(createThingyDTO)
      _        <- EitherT(
                        producer
                        .produce(
                            ThingyCreateRequestedKey(response.id),
                            ThingyCreateRequested(response.body)
                        )
                        .attempt
                        .map {
                            case Right(_) =>
                                ().asRight[HttpErrorDTO]
                            case Left(t) =>
                                InternalServerErrorDTO(msg = s"Failed to produce create event: ${t.getMessage}")
                                    .asLeft[Unit]
                        }
                  )
    yield CreateThingyResponseDTO(response.id)
```

In this case we leverage Cats syntax for [`applicativeError` and `either`](https://github.com/SergeiIonin/ContractsRegistry/blob/master/rest-api/src/main/scala/serverendpoints/CreateContractServerEndpoints.scala).
Additionally, we can write an extension for this:
```scala
TODO
```

### Conclusion
`EitherT` and `Tapir` can be easily paired together to implement server endpoints, developer should take care about provisioning of http-facing dependencies with `EitherT` effect type and write extension to lift `F[_]` into `EitherT` scope to adhere custom DTO types.
