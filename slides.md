---
info: Slides for presentation at Functional Scala 2023
theme: 'default'
presenter: false
download: false
exportFilename: 'presentation'
export:
  format: pdf
  timeout: 30000
  dark: auto
  withClicks: true
  withToc: true
highlighter: 'shiki'
lineNumbers: true
monaco: false
remoteAssets: true
selectable: true
record: false
colorSchema: 'dark'
aspectRatio: '16/9'
canvasWidth: 980
favicon: 'functionalScalaIcon.png'
drawings:
  enabled: false
  persist: false
  presenterOnly: false
  syncAll: true

transition: slide-left
background: /conquer.jpg
---

### Elevate and Conquer: Unleashing the Power of High-Level Endpoints in Scala 

<b>December 2023</b>

<div class="absolute top-10 right-16">
  <img src="/functionalScalaLogo.png" class="h-10" />
</div>

---
transition: slide-left
layout: image-right
image: /yo.jpg
class: "flex items-center text-2xl"
---

<div>
  <p>Jorge Vásquez</p>
  <p>Scala Developer</p>
  <p><b>@Ziverge</b></p>
</div>


---
transition: slide-left
layout: image
image: /laptop.jpg
class: "justify-end"
---

# Introduction

---
transition: slide-left
layout: default
---

## **Introduction**

<div class="mt-4 flex h-4/5 w-full items-center gap-5 text-justify">
  <div class="w-2/3">
    <ul>
      <li v-click>Traditionally, developers have relied on constructing <b>routing tables</b> to build their REST APIs</li>
      <li v-click>These tables map specific <b>routes</b> to corresponding <b>request handlers</b></li>
      <li v-click>This involves <b>manual decoding</b> of headers, query parameters and request bodies</li>
      <li v-click>It also involves <b>manual encoding</b> of responses</li>
    </ul>
  </div>
  <div v-click><img src="/lowLevel.jpg" class="h-60"/></div>
</div>


---
transition: slide-left
layout: image
image: /laptop.jpg
class: "justify-end text-right"
---

## Problem: <br/>Launching a **Web Service**<br/> using a **Low-Level approach**

---
transition: slide-left
layout: image-right
image: /routes.jpg
---

## Example: <br/> **Shopping cart API**


<div class="mt-4 flex h-3/5 w-full items-center">
  <ul>
    <li v-click><b>Initialize cart:</b> <code>POST /cart/{userId}</code></li>
    <li v-click><b>Add item:</b> <code>POST /cart/{userId}/item</code></li>
    <li v-click><b>Delete item:</b> <code>DELETE /cart/{userId}/item/{itemId}</code></li>
    <li v-click><b>Update item:</b> <code>PUT /cart/{userId}/item/{itemId}</code></li>
    <li v-click><b>Get cart contents:</b> <code>GET /cart/{userId}</code></li>
  </ul>
</div>

---
transition: slide-left
layout: default
---

## Shopping Cart using the **ZIO-HTTP Routes API**

<div class="flex h-4/5 w-full items-center">
```scala {all} {maxHeight:'400px'}
// build.sbt
libraryDependencies ++= Seq(
  "dev.zio" %% "zio-http"   % zioHttpVersion
)
```
</div>

<style>
  .slidev-code-wrapper {
    @apply w-full
  }
</style>

---
transition: slide-left
layout: default
---

## Shopping Cart using the **ZIO-HTTP Routes API**

```scala {1|3|4|6-9|11-18|20-37|39-43} {maxHeight:'400px'}
import zio.json._

// Step 1: Define Models
type UserId = UUID

type ItemId = UUID
implicit val encoder: JsonFieldEncoder[UUID] = JsonFieldEncoder.string.contramap(_.toString())
implicit val decoder: JsonFieldDecoder[UUID] =
  JsonFieldDecoder.string.mapOrFail(str => Try(UUID.fromString(str)).toEither.left.map(_.getMessage()))

final case class Item(id: ItemId, name: String, price: Double, quantity: Int) {
  self =>
  def withQuantity(quantity: Int): Item = self.copy(quantity = quantity)
}

object Item {
  implicit val jsonCodec: JsonCodec[Item] = DeriveJsonCodec.gen[Item]
}

final case class Items(items: Map[ItemId, Item]) {
  self =>

  def +(item: Item): Items = Items(self.items + (item.id -> item))

  def -(itemId: ItemId): Items = Items(self.items - itemId)

  def take(n: Int): Items = Items(self.items.take(n))

  def updateQuantity(itemId: ItemId, quantity: Int): Items =
    Items(self.items.updatedWith(itemId)(_.map(_.withQuantity(quantity))))
}

object Items {
  val empty = Items(Map.empty)

  implicit val jsonCodec: JsonCodec[Items] = DeriveJsonCodec.gen[Items]
}

final case class UpdateItemRequest(quantity: Int)

object UpdateItemRequest {
  implicit val jsonCodec: JsonCodec[UpdateItemRequest] = DeriveJsonCodec.gen[UpdateItemRequest]
}
```

---
transition: slide-left
layout: default
---

## Shopping Cart using the **ZIO-HTTP Routes API**

```scala {1-3|5|6|7|8|9|10|11|12|6-13} {maxHeight:'400px'}
import zio._
import zio.http._
import zio.json._

// Step 2: Define Routes: Path Segments, Path Variables, Validation and Handlers
val routes =
    Routes(
      Method.POST / "cart" / uuid("userId")                             -> handler(handleInitializeCart _),
      Method.POST / "cart" / uuid("userId") / "item"                    -> handler(handleAddItem _),
      Method.DELETE / "cart" / uuid("userId") / "item" / uuid("itemId") -> handler(handleRemoveItem _),
      Method.PUT / "cart" / uuid("userId") / "item" / uuid("itemId")    -> handler(handleUpdateItem _),
      Method.GET / "cart" / uuid("userId")                              -> handler(handleGetCartContents _)
    )
```

---
transition: slide-left
layout: default
---

<div class="flex w-full h-full justify-center items-center">
  <div><img src="/tooEasy.jpg" class="h-90"/></div>
</div>

---
transition: slide-left
layout: default
---

## Shopping Cart using the **ZIO-HTTP Routes API**

```scala {1-2|4|5-16|18-34|36-41} {maxHeight:'400px'}
import zio._
import zio.macros._

// Step 3: Define a CartService
@accessible
trait CartService {
  def initialize(userId: UserId): UIO[Unit]

  def addItem(userId: UserId, item: Item): UIO[Items]

  def removeItem(userId: UserId, itemId: ItemId): UIO[Items]

  def updateItemQuantity(userId: UserId, itemId: ItemId, quantity: Int): UIO[Items]

  def getContents(userId: UserId): UIO[Items]
}

final case class CartServiceLive(carts: Ref[Map[UserId, Items]]) extends CartService {
  self =>

  def initialize(userId: UserId): UIO[Unit] = self.carts.update(_ + (userId -> Items.empty))

  def addItem(userId: UserId, item: Item): UIO[Items] = self.updateCartsWith(userId)(_ + item)

  def removeItem(userId: UserId, itemId: ItemId): UIO[Items] = self.updateCartsWith(userId)(_ - itemId)

  def updateItemQuantity(userId: UserId, itemId: ItemId, quantity: Int): UIO[Items] =
    self.updateCartsWith(userId)(_.updateQuantity(itemId, quantity))

  def getContents(userId: UserId): UIO[Items] = self.carts.get.map(_.getOrElse(userId, Items.empty))

  private def updateCartsWith(userId: UserId)(f: Items => Items): UIO[Items] =
    self.carts.updateAndGet(_.updatedWith(userId)(_.map(f))).map(_.getOrElse(userId, Items.empty))
}

object CartServiceLive {
  val layer: ULayer[CartService] =
    ZLayer {
      Ref.make(Map.empty[UserId, Items]).map(CartServiceLive(_))
    }
}
```

---
transition: slide-left
layout: default
---

<div class="flex w-full h-full justify-center items-center">
  <div><img src="/classicZio.jpg" class="h-70"/></div>
</div>

---
transition: slide-left
layout: default
---

## Shopping Cart using the **ZIO-HTTP Routes API**

```scala {1-3|5|7-14|7|8|13|16-36|16|17|21-23|25-34|35|38-45|38|39|44|47-60|47|48|52-54|56-59|62-78|62|63|67-76|77} {maxHeight:'400px'}
import zio._
import zio.http._
import zio.json._

// Step 4: Define Handlers

// POST /cart/{userId}
def handleInitializeCart(userId: UserId, req: Request): URIO[CartService, Response] =
  ZIO.logSpan("initializeCart") {
    for {
      _ <- ZIO.logInfo("Initializing cart")
      _ <- CartService.initialize(userId)
    } yield Response.status(Status.NoContent) // We have to manually create an HTTP Response
  } @@ ZIOAspect.annotated("userId", userId.toString())

// POST /cart/{userId}/item
def handleAddItem(userId: UserId, req: Request): URIO[CartService, Response] =
  ZIO.logSpan("addItem") {
    for {
      _ <- ZIO.logInfo("Adding item to cart")
      // We have to manually decode the body as JSON, handling errors
      body   <- req.body.asString.orDie
      item   <- ZIO.fromEither(body.fromJson[Item]).orDieWith(new RuntimeException(_))
      items0 <- CartService.addItem(userId, item)
      // We have to manually obtain and validate headers from the `Request` object
      allItems = req.headers.get("X-ALL-ITEMS")
      items <- allItems match {
                case Some(allItems) =>
                  ZIO.attempt(allItems.toBoolean).orDie.map {
                    case true  => items0
                    case false => Items.empty + item
                  }
                case None => ZIO.succeed(Items.empty + item)
              }
    } yield Response.json(items.toJson) // We have to manually encode the Response as JSON
  } @@ ZIOAspect.annotated("userId", userId.toString())

// DELETE /cart/{userId}/item/{itemId}
def handleRemoveItem(userId: UserId, itemId: ItemId, req: Request): URIO[CartService, Response] =
  ZIO.logSpan("removeItem") {
    for {
      _     <- ZIO.logInfo("Removing item from cart")
      items <- CartService.removeItem(userId, itemId)
    } yield Response.json(items.toJson) // We have to manually encode the Response as JSON
  } @@ ZIOAspect.annotated("userId" -> userId.toString(), "itemId" -> itemId.toString())

// PUT /cart/{userId}/item/{itemId}
def handleUpdateItem(userId: UserId, itemId: ItemId, req: Request): URIO[CartService, Response] =
  ZIO.logSpan("updateItem") {
    for {
      _ <- ZIO.logInfo("Updating item")
      // We have to manually decode the body as JSON, handling errors
      body              <- req.body.asString.orDie
      updateItemRequest <- ZIO.fromEither(body.fromJson[UpdateItemRequest]).orDieWith(new RuntimeException(_))
      items             <- CartService.updateItemQuantity(userId, itemId, updateItemRequest.quantity)
    } yield Response(
      Status.Ok,
      Headers(Header.ContentType(MediaType.application.json))
    ) // We have to manually set the Response content type
  } @@ ZIOAspect.annotated("userId" -> userId.toString(), "itemId" -> itemId.toString())

// GET /cart/{userId}
def handleGetCartContents(userId: UserId, req: Request): URIO[CartService, Response] =
  ZIO.logSpan("getCartContents") {
    for {
      _ <- ZIO.logInfo("Getting cart contents")
      // We have to manually obtain and validate query parameters from the `Request` object
      limit = req.url.queryParams.get("limit").flatMap(_.headOption)
      items <- limit match {
                case Some(limit) =>
                  ZIO
                    .attempt(limit.toInt)
                    .orDie
                    .flatMap(limit => CartService.getContents(userId).map(_.take(limit)))
                case None => CartService.getContents(userId)
              }
    } yield Response.json(items.toJson) // We have to manually encode the Response as JSON
  } @@ ZIOAspect.annotated("userId" -> userId.toString())
```

---
transition: slide-left
layout: default
---

<div class="flex w-full h-full justify-center items-center">
  <div><img src="/boring.jpg" class="h-70"/></div>
</div>

--
transition: slide-left
layout: default
---

<div class="flex w-full h-full justify-center items-center">
  <div><img src="/nothingFree.jpg" class="h-70"/></div>
</div>

---
transition: slide-left
layout: quote
---

# Can we do **better?**

---
transition: slide-left
layout: default
---

<div class="flex w-full h-full justify-center items-center">
  <div><img src="/picardOfCourse.jpg" class="h-70"/></div>
</div>

---
transition: slide-left
layout: image
image: /theater.jpg
class: "text-center justify-center"
---

# Use High-Level<br/>Endpoint Libraries!

---
transition: slide-left
layout: default
---

## Use High-Level **Endpoint** Libraries!

<div class="mt-4 flex h-4/5 w-full items-center gap-5 text-justify">
  <div class="w-2/3">
    <ul>
      <li v-click>The Scala open-source ecosystem offers <b>superior alternatives</b> for API development</li>
      <li v-click>Libraries like <b>Endpoints4s</b>, <b>Tapir</b> and <b>ZIO HTTP</b> enable developers to define Endpoints at a higher level</li>
      <li v-click>They <b>eliminate</b> the need to handle decoding and encoding manually</li>
      <li v-click>They provide benefits like <b>OpenAPI documentation</b> and <b>type-safe clients</b> totally for free</li>
    </ul>
  </div>
  <div v-click><img src="/highLevel.jpeg" class="h-60"/></div>
</div>

---
transition: slide-left
layout: default
---

## Use High-Level **Endpoint** Libraries!

<div class="flex w-full h-full justify-center items-center">
  <div><img src="/sourceTruth.jpeg" class="h-60"/></div>
</div>

---
transition: slide-left
layout: image
image: /laptop.jpg
class: "justify-end text-right"
---

## Example:<br/>Shopping Cart using **Endpoints4s**

---
transition: slide-left
layout: default
---

## Shopping Cart using **Endpoints4s**

<div class="flex h-4/5 w-full items-center">
```scala {all|3|4|5|6|7|8|9|10} {maxHeight:'400px'}
// build.sbt
libraryDependencies ++= Seq(
  "org.endpoints4s"               %% "algebra"                 % endpoints4sVersion,
  "org.endpoints4s"               %% "json-schema-generic"     % endpoints4sVersion,
  "org.endpoints4s"               %% "http4s-server"           % endpoints4sHttp4sServerVersion,
  "org.endpoints4s"               %% "http4s-client"           % endpoints4sHttp4sClientVersion,
  "org.endpoints4s"               %% "openapi"                 % endpoints4sOpenApiVersion,
  "org.http4s"                    %% "http4s-blaze-server"     % http4sBlazeVersion,
  "org.http4s"                    %% "http4s-blaze-client"     % http4sBlazeVersion,
  "dev.zio"                       %% "zio-interop-cats"        % zioInteropCatsVersion
)
```
</div>

<style>
  .slidev-code-wrapper {
    @apply w-full
  }
</style>

---
transition: slide-left
layout: default
---

## Shopping Cart using **Endpoints4s**

```scala {1-4|6|7|9|11|13-21|23-38|40} {maxHeight:'400px'}
import endpoints4s.generic.docs
import zio._

import java.util.UUID

// Step 1: Define Models
type Eff[+A] = RIO[CartService, A]

type ItemId = UUID

type UserId = UUID

final case class Item(
  @docs("The Item's unique identifier") id: ItemId,
  @docs("The Item's name") name: String,
  @docs("The Item's unit price") price: Double,
  @docs("The Item's quantity") quantity: Int
) {
  self =>
  def withQuantity(quantity: Int): Item = self.copy(quantity = quantity)
}

final case class Items(items: Map[ItemId, Item]) {
  self =>

  def +(item: Item): Items = Items(self.items + (item.id -> item))

  def -(itemId: ItemId): Items = Items(self.items - itemId)

  def take(n: Int): Items = Items(self.items.take(n))

  def updateQuantity(itemId: ItemId, quantity: Int): Items =
    Items(self.items.updatedWith(itemId)(_.map(_.withQuantity(quantity))))
}

object Items {
  val empty = Items(Map.empty)
}

final case class UpdateItemRequest(@docs("The new item quantity") quantity: Int)
```

---
transition: slide-left
layout: default
---

## Shopping Cart using **Endpoints4s**

```scala {1-7|9|10|11|12|13|14|16|18-23|25|27|28|29|31-36|33|34|35|38-54|41|42|43-50|52|53|56-61|63-68|70-74|71} {maxHeight:'400px'}
import endpoints4s.{ algebra, generic }
import endpoints4s.Validated

import java.util.UUID

import scala.util.Success
import scala.util.Try

// Step 2: Define Endpoints
trait Endpoints
    extends algebra.Endpoints
    with algebra.JsonEntitiesFromSchemas
    with generic.JsonSchemas
    with algebra.StatusCodes {

  implicit lazy val itemSchema: JsonSchema[Item] = genericJsonSchema

  implicit lazy val itemsSchema: JsonSchema[Items] =
    mapJsonSchema[Item].xmapPartial(values =>
      Validated
        .traverse(values.toList) { case (k, v) => Validated.fromTry(Try(UUID.fromString(k))).map(_ -> v) }
        .map(items => Items(items.toMap))
    )(_.items.map { case (k, v) => (k.toString(), v) }.toMap)

  implicit lazy val updateItemRequestSchema: JsonSchema[UpdateItemRequest] = genericJsonSchema

  val userId = segment[UUID]("userId", docs = Some("The unique identifier of a user"))
  val itemId = segment[UUID]("itemId", docs = Some("The unique identifier of an item"))
  val limit  = qs[Option[Int]]("limit", docs = Some("The maximum number of items to obtain"))

  val initializeCart =
    endpoint(
      post(path / "cart" / userId, emptyRequest),
      response(NoContent, emptyResponse),
      docs = EndpointDocs().withDescription(Some("Initiliaze a user's cart"))
    )

  val addItem =
    endpoint(
      post(
        path / "cart" / userId / "item",
        jsonRequest[Item],
        headers = optRequestHeader(
          "X-ALL-ITEMS",
          docs = Some("Flag to indicate whether to return all items or just the new one")
        ).xmapPartial(allItems =>
          allItems.fold(Validated.fromTry(Success(Option.empty[Boolean])))(allItems =>
            Validated.fromOption(allItems.toBooleanOption)("invalid value").map(Some(_))
          )
        )(_.map(_.toString))
      ),
      ok(jsonResponse[Items], docs = Some("The operation result")),
      docs = EndpointDocs().withDescription(Some("Add an item to a user's cart"))
    )

  val removeItem =
    endpoint(
      delete(path / "cart" / userId / "item" / itemId),
      ok(jsonResponse[Items], docs = Some("The cart items after removal")),
      docs = EndpointDocs().withDescription(Some("Removes an item from a user's cart"))
    )

  val updateItem =
    endpoint(
      put(path / "cart" / userId / "item" / itemId, jsonRequest[UpdateItemRequest], docs = Some("The request object")),
      ok(jsonResponse[Items], docs = Some("The cart items after updating")),
      docs = EndpointDocs().withDescription(Some("Updates an item"))
    )

  val getCartContents = endpoint(
    get(path / "cart" / userId /? limit),
    ok(jsonResponse[Items], docs = Some("The cart items")),
    docs = EndpointDocs().withDescription(Some("Gets the contents of a user's cart"))
  )
}
```

---
transition: slide-left
layout: default
---

## Shopping Cart using **Endpoints4s**

```scala {1-4|6|7|8|9|10|11|13-20|13|22-33|35-42|44-51|53-60|62-64|66-73} {maxHeight:'400px'}
import org.http4s._
import org.http4s.blaze.server.BlazeServerBuilder
import zio._
import zio.interop.catz._

// Step 3: Generate Http4s server, backed by ZIO, from Endpoints
object Endpoints4sServer
    extends endpoints4s.http4s.server.Endpoints[Eff]
    with endpoints4s.http4s.server.JsonEntitiesFromSchemas
    with Endpoints
    with ZIOAppDefault {

  val initializeCartRoute = initializeCart.implementedByEffect { userId =>
    ZIO.logSpan("initializeCart") {
      for {
        _ <- ZIO.logInfo("Initializing cart")
        _ <- CartService.initialize(userId)
      } yield ()
    } @@ ZIOAspect.annotated("userId", userId.toString())
  }

  val addItemRoute = addItem.implementedByEffect { case ((userId, item, allItems)) =>
    ZIO.logSpan("addItem") {
      for {
        _      <- ZIO.logInfo("Adding item to cart")
        items0 <- CartService.addItem(userId, item)
        items   = allItems match {
                    case Some(true) => items0
                    case _          => Items.empty + item
                  }
      } yield items
    } @@ ZIOAspect.annotated("userId", userId.toString())
  }

  val removeItemRoute = removeItem.implementedByEffect { case (userId, itemId) =>
    ZIO.logSpan("removeItem") {
      for {
        _     <- ZIO.logInfo("Removing item from cart")
        items <- CartService.removeItem(userId, itemId)
      } yield items
    } @@ ZIOAspect.annotated("userId" -> userId.toString(), "itemId" -> itemId.toString())
  }

  val updateItemRoute = updateItem.implementedByEffect { case (userId, itemId, updateItemRequest) =>
    ZIO.logSpan("updateItem") {
      for {
        _     <- ZIO.logInfo("Updating item")
        items <- CartService.updateItemQuantity(userId, itemId, updateItemRequest.quantity)
      } yield items
    } @@ ZIOAspect.annotated("userId" -> userId.toString(), "itemId" -> itemId.toString())
  }

  val getCartContentsRoute = getCartContents.implementedByEffect { case (userId, limit) =>
    ZIO.logSpan("getCartContents") {
      for {
        _     <- ZIO.logInfo("Getting cart contents")
        items <- CartService.getContents(userId)
      } yield limit.fold(items)(items.take)
    } @@ ZIOAspect.annotated("userId" -> userId.toString())
  }

  val routes: HttpRoutes[Eff] = HttpRoutes.of {
    routesFromEndpoints(initializeCartRoute, addItemRoute, removeItemRoute, updateItemRoute, getCartContentsRoute)
  }

  override val run =
    BlazeServerBuilder[Eff]
      .bindHttp(8080, "localhost")
      .withHttpApp(routes.orNotFound)
      .serve
      .compile
      .drain
      .provide(CartServiceLive.layer)
}
```

---
transition: slide-left
layout: default
---

<div class="flex w-full h-full justify-center items-center">
  <div><img src="/thereIsMore.jpg" class="h-70"/></div>
</div>

---
transition: slide-left
layout: default
---

## Bonus: Shopping Cart Client using **Endpoints4s**

```scala {1-11|13|15|18|19-29|30-32|33|34-36|37-39|40|41|42-44|45} {maxHeight:'400px'}
import com.comcast.ip4s.IpAddress
import com.comcast.ip4s.Ipv4Address
import org.http4s._
import org.http4s.blaze.client.BlazeClientBuilder
import org.http4s.Uri.Authority
import org.http4s.Uri.Host
import org.http4s.Uri.Scheme
import zio._
import zio.interop.catz._

import java.util.UUID

object Endpoints4sClient extends ZIOAppDefault {

  val clientExample =
    ZIO.scoped {
      for {
        // Instantiate an Http4sClient backed by ZIO
        client  <- BlazeClientBuilder[Eff].resource.toScopedZIO.map { http4sClient =>
                     new endpoints4s.http4s.client.Endpoints(
                       Authority(
                         None,
                         Host.fromIpAddress(IpAddress.fromString("127.0.0.1").get),
                         Some(8080)
                       ),
                       Scheme.http,
                       http4sClient
                     ) with endpoints4s.http4s.client.JsonEntitiesFromSchemas with Endpoints
                   }
        userId  <- ZIO.succeed(UUID.randomUUID())
        itemId1 <- ZIO.succeed(UUID.randomUUID())
        itemId2 <- ZIO.succeed(UUID.randomUUID())
        _       <- client.initializeCart.sendAndConsume(userId)
        _       <- client.addItem
                     .sendAndConsume((userId, Item(itemId1, "test-item-1", 10.0, 10), None))
                     .debug("addItem result")
        _       <- client.addItem
                     .sendAndConsume(userId, Item(itemId2, "test-item-2", 20.0, 20), Some(true))
                     .debug("addItem result")
        _       <- client.removeItem.sendAndConsume(userId, itemId2).debug("removeItem result")
        _       <- client.updateItem.sendAndConsume(userId, itemId1, UpdateItemRequest(35)).debug("updateItem result")
        _       <- client.addItem
                     .sendAndConsume(userId, Item(itemId2, "test-item-2", 20.0, 20), Some(true))
                     .debug("addItem result")
        _       <- client.getCartContents.sendAndConsume(userId, Some(1)).debug("getCartContents result")
      } yield ()
    }

  val run = clientExample.provide(CartServiceLive.layer)
}
```

---
transition: slide-left
layout: default
---

<div class="flex w-full h-full justify-center items-center">
  <div><img src="/stillMore.jpg" class="h-70"/></div>
</div>

---
transition: slide-left
layout: default
---

## Bonus: Shopping Cart Docs using **Endpoints4s**

```scala {1-2|4|5|6|7|8|10|12-19|21|23-24} {maxHeight:'400px'}
import endpoints4s.openapi
import zio._

object Endpoints4sDocs
    extends openapi.Endpoints
    with openapi.JsonEntitiesFromSchemas
    with Endpoints
    with ZIOAppDefault {

  import openapi.model._

  val docs =
    openApi(Info(title = "Shopping cart", version = "0.1.0"))(
      initializeCart,
      addItem,
      removeItem,
      updateItem,
      getCartContents
    )

  val docsJson = OpenApi.stringEncoder.encode(docs)

  override val run =
    Console.printLine(s"OpenAPI docs:\n$docsJson")
}
```

---
transition: slide-left
layout: image
image: /laptop.jpg
class: "justify-end text-right"
---

## Example:<br/>Shopping Cart using **Tapir**

---
transition: slide-left
layout: default
---

## Shopping Cart using **Tapir**

<div class="flex h-4/5 w-full items-center">
```scala {all|3|4|5|6|7|8|9|10|11} {maxHeight:'400px'}
// build.sbt
libraryDependencies ++= Seq(
  "com.softwaremill.sttp.tapir"   %% "tapir-core"              % tapirVersion,
  "com.softwaremill.sttp.tapir"   %% "tapir-http4s-server-zio" % tapirVersion,
  "com.softwaremill.sttp.tapir"   %% "tapir-json-zio"          % tapirVersion,
  "com.softwaremill.sttp.tapir"   %% "tapir-sttp-client"       % tapirVersion,
  "com.softwaremill.sttp.tapir"   %% "tapir-openapi-docs"      % tapirVersion,
  "com.softwaremill.sttp.apispec" %% "openapi-circe-yaml"      % openApiCirceYamlVersion,
  "com.softwaremill.sttp.client3" %% "zio"                     % sttpZioVersion,
  "org.http4s"                    %% "http4s-blaze-server"     % http4sBlazeVersion,
  "dev.zio"                       %% "zio-interop-cats"        % zioInteropCatsVersion
)
```
</div>

<style>
  .slidev-code-wrapper {
    @apply w-full
  }
</style>

---
transition: slide-left
layout: default
---

## Shopping Cart using **Tapir**

```scala {1-8|9|10|12-15|17|19-32|34-55|57-61} {maxHeight:'400px'}
import sttp.tapir.Schema.annotations.description
import sttp.tapir.Schema
import zio._
import zio.json._

import scala.util.Try
import java.util.UUID

// Step 1: Define Models
type Eff[+A] = RIO[CartService, A]

type ItemId = UUID
implicit val encoder: JsonFieldEncoder[UUID] = JsonFieldEncoder.string.contramap(_.toString())
implicit val decoder: JsonFieldDecoder[UUID] =
  JsonFieldDecoder.string.mapOrFail(str => Try(UUID.fromString(str)).toEither.left.map(_.getMessage()))

type UserId = UUID

final case class Item(
  @description("The Item's unique identifier") id: ItemId,
  @description("The Item's name") name: String,
  @description("The Item's unit price") price: Double,
  @description("The Item's quantity") quantity: Int
) {
  self =>
  def withQuantity(quantity: Int): Item = self.copy(quantity = quantity)
}

object Item {
  implicit val jsonCodec: JsonCodec[Item] = DeriveJsonCodec.gen[Item]
  implicit val schema: Schema[Item]       = Schema.derived[Item]
}

final case class Items(items: Map[ItemId, Item]) {
  self =>

  def +(item: Item): Items = Items(self.items + (item.id -> item))

  def -(itemId: ItemId): Items = Items(self.items - itemId)

  def take(n: Int): Items = Items(self.items.take(n))

  def updateQuantity(itemId: ItemId, quantity: Int): Items =
    Items(self.items.updatedWith(itemId)(_.map(_.withQuantity(quantity))))
}

object Items {
  val empty = Items(Map.empty)

  implicit val jsonCodec: JsonCodec[Items] = DeriveJsonCodec.gen[Items]
  implicit val schema: Schema[Items]       = Schema
    .schemaForMap[ItemId, Item](_.toString())
    .map(items => Some(Items(items)))(_.items)
    .description("Map of item IDs to corresponding items")
}

final case class UpdateItemRequest(@description("The new item quantity") quantity: Int)
object UpdateItemRequest {
  implicit val jsonCodec: JsonCodec[UpdateItemRequest] = DeriveJsonCodec.gen[UpdateItemRequest]
  implicit val schema: Schema[UpdateItemRequest]       = Schema.derived[UpdateItemRequest]
}
```

---
transition: slide-left
layout: default
---

## Shopping Cart using **Tapir**

```scala {1-6|7|8|9|10|11|12-13|15-19|16|17|18|19|21-27|22|23|24|25|26|27|29-33|30|31|32|33|35-40|36|37|38|39|40|42-47|43|44|45|46|47} {maxHeight:'400px'}
import sttp.model.StatusCode
import sttp.tapir.json.zio._
import sttp.tapir.ztapir._

import java.util.UUID

// Step 2: Define Endpoints
trait Endpoints {
  val userId    = path[UUID]("userId").description("The unique identifier of a user")
  val itemId    = path[UUID]("userId").description("The unique identifier of an item")
  val limit     = query[Option[Int]]("limit").description("The maximum number of items to obtain")
  val xAllItems = header[Option[Boolean]]("X-ALL-ITEMS")
    .description("Flag to indicate whether to return all items or just the new one")

  val initializeCart =
    endpoint.post
      .in("cart" / userId)
      .out(statusCode(StatusCode.NoContent))
      .description("Initiliaze a user's cart")

  val addItem =
    endpoint.post
      .in("cart" / userId / "item")
      .in(xAllItems)
      .in(jsonBody[Item].description("The item to be added"))
      .out(jsonBody[Items].description("The operation result"))
      .description("Add an item to a user's cart")

  val removeItem =
    endpoint.delete
      .in("cart" / userId / "item" / itemId)
      .out(jsonBody[Items].description("The cart items after removal"))
      .description("Removes an item from a user's cart")

  val updateItem =
    endpoint.put
      .in("cart" / userId / "item" / itemId)
      .in(jsonBody[UpdateItemRequest].description("The request object"))
      .out(jsonBody[Items].description("The cart items after updating"))
      .description("Updates an item")

  val getCartContents =
    endpoint.get
      .in("cart" / userId)
      .in(limit)
      .out(jsonBody[Items].description("The cart items"))
      .description("Gets the contents of a user's cart")
}
```

---
transition: slide-left
layout: default
---

## Shopping Cart using **Tapir**

```scala {1-6|7|8|9-17|19-30|32-39|41-48|50-57|59-70|72-79} {maxHeight:'400px'}
import sttp.tapir.server.http4s.ztapir.ZHttp4sServerInterpreter
import org.http4s.blaze.server.BlazeServerBuilder
import org.http4s.HttpRoutes
import zio._
import zio.interop.catz._

// Step 3: Generate Http4s server, backed by ZIO, from Endpoints
object TapirServer extends ZIOAppDefault with Endpoints {

  val initializeCartLogic = initializeCart.zServerLogic { userId =>
    ZIO.logSpan("initializeCart") {
      for {
        _ <- ZIO.logInfo("Initializing cart")
        _ <- CartService.initialize(userId)
      } yield ()
    } @@ ZIOAspect.annotated("userId", userId.toString())
  }

  val addItemLogic = addItem.zServerLogic { case (userId, allItems, item) =>
    ZIO.logSpan("addItem") {
      for {
        _      <- ZIO.logInfo("Adding item to cart")
        items0 <- CartService.addItem(userId, item)
        items   = allItems match {
                    case Some(true) => items0
                    case _          => Items.empty + item
                  }
      } yield items
    } @@ ZIOAspect.annotated("userId", userId.toString())
  }

  val removeItemLogic = removeItem.zServerLogic { case (userId, itemId) =>
    ZIO.logSpan("removeItem") {
      for {
        _     <- ZIO.logInfo("Removing item from cart")
        items <- CartService.removeItem(userId, itemId)
      } yield items
    } @@ ZIOAspect.annotated("userId" -> userId.toString(), "itemId" -> itemId.toString())
  }

  val updateItemLogic = updateItem.zServerLogic { case (userId, itemId, updateItemRequest) =>
    ZIO.logSpan("updateItem") {
      for {
        _     <- ZIO.logInfo("Updating item")
        items <- CartService.updateItemQuantity(userId, itemId, updateItemRequest.quantity)
      } yield items
    } @@ ZIOAspect.annotated("userId" -> userId.toString(), "itemId" -> itemId.toString())
  }

  val getCartContentsLogic = getCartContents.zServerLogic { case (userId, limit) =>
    ZIO.logSpan("getCartContents") {
      for {
        _     <- ZIO.logInfo("Getting cart contents")
        items <- CartService.getContents(userId)
      } yield limit.fold(items)(items.take)
    } @@ ZIOAspect.annotated("userId" -> userId.toString())
  }

  val routes: HttpRoutes[Eff] =
    ZHttp4sServerInterpreter()
      .from(
        List(
          initializeCartLogic,
          addItemLogic,
          removeItemLogic,
          updateItemLogic,
          getCartContentsLogic
        )
      )
      .toRoutes

  override val run =
    BlazeServerBuilder[Eff]
      .bindHttp(8080, "localhost")
      .withHttpApp(routes.orNotFound)
      .serve
      .compile
      .drain
      .provide(CartServiceLive.layer)
}
```

---
transition: slide-left
layout: default
---

<div class="flex w-full h-full justify-center items-center">
  <div><img src="/bonuses.jpg" class="h-70"/></div>
</div>

---
transition: slide-left
layout: default
---

## Bonus: Shopping Cart Client using **Tapir**

```scala {1-6|8|12|13|14|15-18|19-21|22|23|24|25|26|27|28} {maxHeight:'400px'}
import sttp.model.Uri._
import sttp.tapir.client.sttp.SttpClientInterpreter
import sttp.client3.httpclient.zio.HttpClientZioBackend
import zio._

import java.util.UUID

object TapirClient extends ZIOAppDefault with Endpoints {

  val run =
    for {
      backend              <- HttpClientZioBackend()
      uri                   = Some(uri"http://localhost:8080")
      initializeCartClient  = SttpClientInterpreter().toClient(initializeCart, uri, backend)
      addItemClient         = SttpClientInterpreter().toClient(addItem, uri, backend)
      removeItemClient      = SttpClientInterpreter().toClient(removeItem, uri, backend)
      updateItemClient      = SttpClientInterpreter().toClient(updateItem, uri, backend)
      getCartContentsClient = SttpClientInterpreter().toClient(getCartContents, uri, backend)
      userId               <- ZIO.succeed(UUID.randomUUID())
      itemId1              <- ZIO.succeed(UUID.randomUUID())
      itemId2              <- ZIO.succeed(UUID.randomUUID())
      _                    <- initializeCartClient(userId)
      _                    <- addItemClient((userId, None, Item(itemId1, "test-item-1", 10.0, 10))).debug("addItem result")
      _                    <- addItemClient((userId, Some(true), Item(itemId2, "test-item-2", 20.0, 20))).debug("addItem result")
      _                    <- removeItemClient((userId, itemId2)).debug("removeItem result")
      _                    <- updateItemClient((userId, itemId1, UpdateItemRequest(35))).debug("updateItem result")
      _                    <- addItemClient((userId, Some(true), Item(itemId2, "test-item-2", 20.0, 20))).debug("addItem result")
      _                    <- getCartContentsClient((userId, Some(1))).debug("getCartContents result")
    } yield ()
}
```

---
transition: slide-left
layout: default
---

## Bonus: Shopping Cart Docs using **Tapir**

```scala {1-3|5|7-11|8|9|10|13-14} {maxHeight:'400px'}
import sttp.apispec.openapi.circe.yaml._
import sttp.tapir.docs.openapi.OpenAPIDocsInterpreter
import zio._

object TapirDocs extends ZIOAppDefault with Endpoints {

  val docs = OpenAPIDocsInterpreter().toOpenAPI(
    List(initializeCart, addItem, removeItem, updateItem, getCartContents),
    "Shopping cart",
    "0.1.0"
  )

  override val run =
    Console.printLine(s"OpenAPI docs:\n${docs.toYaml}")
}
```

---
transition: slide-left
layout: image
image: /laptop.jpg
class: "justify-end text-right"
---

## Example:<br/>Shopping Cart using **ZIO-HTTP Endpoints API**

---
transition: slide-left
layout: default
---

## Shopping Cart using the **ZIO-HTTP Endpoints API**

<div class="flex h-4/5 w-full items-center">
```scala {all} {maxHeight:'400px'}
// build.sbt
libraryDependencies ++= Seq(
  "dev.zio" %% "zio-http" % zioHttpVersion
)
```
</div>

<style>
  .slidev-code-wrapper {
    @apply w-full
  }
</style>

---
transition: slide-left
layout: default
---

## Shopping Cart using **ZIO-HTTP Endpoints API**

```scala {1-6|8|9-12|14|16-29|31-48|50-55} {maxHeight:'400px'}
import zio.json._
import zio.schema._
import zio.schema.annotation.description

import scala.util.Try
import java.util.UUID

// Step 1: Define Models
type ItemId = UUID
implicit val encoder: JsonFieldEncoder[UUID] = JsonFieldEncoder.string.contramap(_.toString())
implicit val decoder: JsonFieldDecoder[UUID] =
  JsonFieldDecoder.string.mapOrFail(str => Try(UUID.fromString(str)).toEither.left.map(_.getMessage()))

type UserId = UUID

final case class Item(
  @description("The Item's unique identifier") id: ItemId,
  @description("The Item's name") name: String,
  @description("The Item's unit price") price: Double,
  @description("The Item's quantity") quantity: Int
) {
  self =>
  def withQuantity(quantity: Int): Item = self.copy(quantity = quantity)
}

object Item {
  implicit val jsonCodec: JsonCodec[Item] = DeriveJsonCodec.gen[Item]
  implicit val schema: Schema[Item]       = DeriveSchema.gen[Item]
}

final case class Items(@description("Map of item IDs to corresponding items") items: Map[ItemId, Item]) {
  self =>

  def +(item: Item): Items = Items(self.items + (item.id -> item))

  def -(itemId: ItemId): Items = Items(self.items - itemId)

  def take(n: Int): Items = Items(self.items.take(n))

  def updateQuantity(itemId: ItemId, quantity: Int): Items =
    Items(self.items.updatedWith(itemId)(_.map(_.withQuantity(quantity))))
}
object Items {
  val empty = Items(Map.empty)

  implicit val jsonCodec: JsonCodec[Items] = DeriveJsonCodec.gen[Items]
  implicit val schema: Schema[Items]       = DeriveSchema.gen[Items]
}

final case class UpdateItemRequest(@description("The new item quantity") quantity: Int)

object UpdateItemRequest {
  implicit val jsonCodec: JsonCodec[UpdateItemRequest] = DeriveJsonCodec.gen[UpdateItemRequest]
  implicit val schema: Schema[UpdateItemRequest]       = DeriveSchema.gen[UpdateItemRequest]
}
```

---
transition: slide-left
layout: default
---

## Shopping Cart using **ZIO-HTTP Endpoints API**

```scala {1-4|6|7|8|9|10|11-12|14-16|15|16|18-22|19|20|21|22|24-26|25|26|28-31|29|30|31|33-36|34|35|36} {maxHeight:'400px'}
import zio.http._
import zio.http.codec._
import zio.http.codec.HttpCodec._
import zio.http.endpoint.Endpoint

// Step 2: Define Endpoints
trait Endpoints {
  val userId    = uuid("userId") ?? Doc.p("The unique identifier of a user")
  val itemId    = uuid("itemId") ?? Doc.p("The unique identifier of an item")
  val limit     = paramInt("limit").optional ?? Doc.p("The maximum number of items to obtain")
  val xAllItems = HeaderCodec.name[Boolean]("X-ALL-ITEMS").optional ??
    Doc.p("Flag to indicate whether to return all items or just the new one")

  val initializeCart =
    Endpoint(Method.POST / "cart" / userId)
      .out[Unit](Status.NoContent) ?? Doc.p("Initiliaze a user's cart")

  val addItem =
    Endpoint(Method.POST / "cart" / userId / "item")
      .header(xAllItems)
      .in[Item](Doc.p("The item to be added"))
      .out[Items](Doc.p("The operation result")) ?? Doc.p("Add an item to a user's cart")

  val removeItem =
    Endpoint(Method.DELETE / "cart" / userId / "item" / itemId)
      .out[Items](Doc.p("The cart items after removal")) ?? Doc.p("Removes an item from a user's cart")

  val updateItem =
    Endpoint(Method.PUT / "cart" / userId / "item" / itemId)
      .in[UpdateItemRequest](Doc.p("The request object"))
      .out[Items](Doc.p("The cart items after updating")) ?? Doc.p("Updates an item")

  val getCartContents =
    Endpoint(Method.GET / "cart" / userId)
      .query(limit)
      .out[Items](Doc.p("The cart items")) ?? Doc.p("Gets the contents of a user's cart")
}
```

---
transition: slide-left
layout: default
---

## Shopping Cart using **ZIO-HTTP Endpoints API**

```scala {1-2|4|5|7-13|15-25|27-33|35-41|43-49|51-58|60} {maxHeight:'400px'}
import zio._
import zio.http._

// Step 3: Generate zio-http server
object EndpointsServer extends ZIOAppDefault with Endpoints {

  def handleInitializeCart(userId: UserId) =
    ZIO.logSpan("initializeCart") {
      for {
        _ <- ZIO.logInfo("Initializing cart")
        _ <- CartService.initialize(userId)
      } yield ()
    } @@ ZIOAspect.annotated("userId", userId.toString())

  def handleAddItem(userId: UserId, allItems: Option[Boolean], item: Item) =
    ZIO.logSpan("addItem") {
      for {
        _      <- ZIO.logInfo("Adding item to cart")
        items0 <- CartService.addItem(userId, item)
        items   = allItems match {
                    case Some(true) => items0
                    case _          => Items.empty + item
                  }
      } yield items
    } @@ ZIOAspect.annotated("userId", userId.toString())

  def handleRemoveItem(userId: UserId, itemId: ItemId) =
    ZIO.logSpan("removeItem") {
      for {
        _     <- ZIO.logInfo("Removing item from cart")
        items <- CartService.removeItem(userId, itemId)
      } yield items
    } @@ ZIOAspect.annotated("userId" -> userId.toString(), "itemId" -> itemId.toString())

  def handleUpdateItem(userId: UserId, itemId: ItemId, updateItemRequest: UpdateItemRequest) =
    ZIO.logSpan("updateItem") {
      for {
        _     <- ZIO.logInfo("Updating item")
        items <- CartService.updateItemQuantity(userId, itemId, updateItemRequest.quantity)
      } yield items
    } @@ ZIOAspect.annotated("userId" -> userId.toString(), "itemId" -> itemId.toString())

  def handleGetCartContents(userId: UserId, limit: Option[Int]) =
    ZIO.logSpan("getCartContents") {
      for {
        _     <- ZIO.logInfo("Getting cart contents")
        items <- CartService.getContents(userId)
      } yield limit.fold(items)(items.take)
    } @@ ZIOAspect.annotated("userId" -> userId.toString())

  val routes =
    Routes(
      initializeCart.implement(handler(handleInitializeCart _)),
      addItem.implement(handler(handleAddItem _)),
      removeItem.implement(handler(handleRemoveItem _)),
      updateItem.implement(handler(handleUpdateItem _)),
      getCartContents.implement(handler(handleGetCartContents _))
    )

  val run = Server.serve(routes.sandbox.toHttpApp).provide(Server.default, CartServiceLive.layer)
}
```

---
transition: slide-left
layout: default
---

<div class="flex w-full h-full justify-center items-center">
  <div><img src="/bonusTime.jpeg" class="h-70"/></div>
</div>

---
transition: slide-left
layout: default
---

## Bonus: Shopping Cart Client using **ZIO-HTTP Endpoints API**

```scala {1-5|7|12|13-15|16|17|18|19|20|21|22|26-31} {maxHeight:'400px'}
import zio._
import zio.http._
import zio.http.endpoint.EndpointExecutor

import java.util.UUID

object EndpointsClient extends ZIOAppDefault with Endpoints {

  val clientExample: URIO[EndpointExecutor[Unit], Unit] =
    ZIO.scoped {
      for {
        executor <- ZIO.service[EndpointExecutor[Unit]]
        userId   <- ZIO.succeed(UUID.randomUUID())
        itemId1  <- ZIO.succeed(UUID.randomUUID())
        itemId2  <- ZIO.succeed(UUID.randomUUID())
        _        <- executor(initializeCart(userId))
        _        <- executor(addItem(userId, None, Item(itemId1, "test-item-1", 10.0, 10))).debug("addItem result")
        _        <- executor(addItem(userId, Some(true), Item(itemId2, "test-item-2", 20.0, 20))).debug("addItem result")
        _        <- executor(removeItem(userId, itemId2)).debug("removeItem result")
        _        <- executor(updateItem(userId, itemId1, UpdateItemRequest(35))).debug("updateItem result")
        _        <- executor(addItem(userId, Some(true), Item(itemId2, "test-item-2", 20.0, 20))).debug("addItem result")
        _        <- executor(getCartContents(userId, Some(1))).debug("getCartContents result")
      } yield ()
    }

  val run =
    clientExample
      .provide(
        EndpointExecutor.make(serviceName = "cart_service"),
        Client.default
      )
}
```

---
transition: slide-left
layout: image-right
image: /summary.jpg
---

## **Summary**

<div class="mt-4 flex h-4/5 w-full items-center">
  <ul>
    <li v-click>Classic approach to building APIs is low-level</li>
    <li v-click>Endpoints offer a higher-level solution</li>
    <li v-click>Each endpoint library has slightly different features and excels at slightly different use cases</li>
  </ul>
</div>

---
transition: slide-left
layout: image-right
image: /learn.jpg
---

## **To learn more...**

<div class="flex h-3/5 w-full items-center">
  <ul>
    <li>Visit the <strong>Tapir</strong> <a href="https://github.com/softwaremill/tapir" target="_blank">GitHub repo</a></li>
    <li>Visit the <strong>Endpoints4s</strong> <a href="https://github.com/endpoints4s/endpoints4s" target="_blank">GitHub repo</a></li>
    <li>Visit the <strong>ZIO HTTP</strong> <a href="https://github.com/zio/zio-http" target="_blank">GitHub repo</a></li>
  </ul>
</div>

---
transition: slide-left
layout: image-right
image: /thanks.jpg
---

## **Special Thanks**

<div class="flex h-3/5 w-full items-center">
  <ul>
    <li><strong>Ziverge</strong> for organizing this conference</li>
    <li><strong>John De Goes</strong> for guidance and support</li>
  </ul>
</div>

---
transition: slide-left
layout: image-right
image: /computer.png
---

## **Contact me**

<div class="grid grid-cols-8 gap-4 items-center h-4/5 content-center text-2xl">
  <div class="col-span-1"><img src="/x.png" class="w-8" /></div> <div class="col-span-7">@jorvasquez2301</div>
  <div class="col-span-1"><img src="/linkedin.png" class="w-8" /></div> <div class="col-span-7">jorge-vasquez-2301</div>
  <div class="col-span-1"><img src="/email.png" class="w-8" /></div> <div class="col-span-7">jorge.vasquez@ziverge.com</div>
</div>