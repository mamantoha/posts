---
title: "Decoding JSON with unknown structure with Elm"
created_at: 2017-11-16 18:00:09 +0100
kind: article
publish: false
author: Anton Paisov
tags: [ 'elm', 'json' ]
newsletter: :skip
---

When you need to write a [decoder](http://package.elm-lang.org/packages/elm-lang/core/5.1.1/Json-Decode) Elm in order to map JSON to a corresponding Model.

<!-- more -->

Paweł and I been working on a mountable web interface for [RailsEventStore](https://railseventstore.org) recently.
The purpose of it is to display events and event streams without launching rails console for that.
This web interface is written in Elm and hopefully will be a part of the next RailsEventStore release.

Now back to the main topic of this post.

For the purpose of this post here's how we imported `Json.Decode`.

```elm
import Json.Decode as Decode exposing (Decoder, map, field, list, string, at, value)
```

Now here's how a basic Elm JSON decoder for our event stream looks like:

```elm
streamDecoder : Decode.Decoder Item
streamDecoder =
    Decode.map Stream
        (field "name" string)
```

It gets more interesting when we want to decode an individual event.
In RailsEventStore user decides what will go in `data` and in `metadata` for a particular type of event.
As a result, there is no schema that we could describe statically and we _just_ want to map `data` and `metadata` as strings in our event decoder.
But you can't pass JSON subtree as a string. Here's the solution we've ended up with:

```elm
getEvent : String -> Cmd Msg
getEvent eventId =
    let
        decoder =
            Decode.andThen eventWithDetailsDecoder rawEventDecoder
    in
        Http.send EventDetails (Http.get "/event.json" decoder)

rawEventDecoder : Decoder ( Decode.Value, Decode.Value )
rawEventDecoder =
    Decode.map2 (,)
        (field "data" value)
        (field "metadata" value)

eventWithDetailsDecoder : ( Decode.Value, Decode.Value ) -> Decode.Decoder EventWithDetails
eventWithDetailsDecoder ( data, metadata ) =
    Decode.map4 EventWithDetails
        (field "event_type" string)
        (field "event_id" string)
        (field "data" (Decode.succeed (toString data)))
        (field "metadata" (Decode.succeed (toString metadata)))

```

The solution is fairly self-explanatory and is possible thanks to [`andThen`](http://package.elm-lang.org/packages/elm-lang/core/5.1.1/Json-Decode#andThen) function which creates decoders that depend on previous results.