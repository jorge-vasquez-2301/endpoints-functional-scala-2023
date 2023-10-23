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

## Implementation using the **ZIO-HTTP Routes API**

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

## Implementation using the **ZIO-HTTP Routes API**

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

## Implementation using the **ZIO-HTTP Routes API**

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

## Implementation using the **ZIO-HTTP Routes API**

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
class: "justify-center"
---

# Use High-Level Endpoints!

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