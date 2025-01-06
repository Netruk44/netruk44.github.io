---
title: "Tutorial: Adding announcements to your SwiftUI app"
date: 2024-11-03T20:55:35-05:00
draft: false
tags: ["Swift", "SwiftUI", "iOS", "macOS", "visionOS", "Apple"]
---

<!-- TODO: Screenshot of the announcement banner in the app -->

I haven't been posting as much as I'd like to be to this blog because I've been working on my [Apple Vision Pro toy](/project/2024-physics-playground/). So I thought I'd write a quick tutorial on how to add announcements to your SwiftUI app based on how I implemented it in my app.

This won't be as exhaustive as some of my previous tutorials because I just wanted to write something quickly. But I hope it's still helpful!

> **Important Note**: As always, I never claim my way to be *the best* way of accomplishing something. I'm still learning and I'm sure there are better ways to do a bunch of this stuff.

{{% toc %}}

## The Overview

The goal: You have an app where you want to be able to show messages to the user. For example, you might want to display a message about a known issue, advertise your TestFlight beta, or maybe just say "Happy Halloween!"

In addition, it would be nice to be able to include links as well as target specific versions of your app (after all, hopefully a known issue will eventually be addressed!).

To accomplish this, we'll write some code that will download a JSON file from a server that contains the announcements. It will construct the URL using information about its version, then parse the returned JSON and display the announcements as needed.

The specifics of how the JSON file gets made won't be covered here. You could handcraft some JSON files that could then be stored on a server, or S3/Azure Blob Storage. Or you could even dynamically generate them using PHP/a serverless function or whatever else your needs desire. The important part is that the JSON file be accessible via a URL.

## The Announcement Model

First, we need a model to represent the announcement. When starting fresh, I find it helpful to start with a made up JSON example to parse. That way the code that needs to be written is clear.

But since I'm already done, here's an example of what the JSON eventually wound up looking like after some revisions. It's an announcement I use for TestFlight beta testers:

```json
{
  "lines": [
    {
      "type": "text",
      "data": {
        "text": "This is a pre-release version of Spatial Physics Playground."
      }
    },
    {
      "type": "text",
      "data": {
        "text": "Thanks for helping me test the upcoming update!"
      }
    },
    {
      "type": "text",
      "data": {
        "text": ""
      }
    },
    {
      "type": "text",
      "data": {
        "text": "If you encounter bugs or have suggestions, use the Feedback tool to let me know."
      }
    },
    {
      "type": "text",
      "data": {
        "text": ""
      }
    },
    {
      "type": "link",
      "data": {
        "text": "Tap here to go back to the currently live App Store version",
        "url": "https://apps.apple.com/us/app/spatial-physics-playground/id6480322995"
      }
    }
  ]
}
```

As you can see, it's a very simple structure, just a list of lines.

There are two different types of line: `text` and `link`. I think they don't really need much explanation.

This is what I came up with:

```swift
struct Announcement: Decodable {
    struct Line: Decodable, Hashable {
        let id: UUID
        let type: String
        let data: [String:String]

        enum CodingKeys: String, CodingKey {
            case type, data
        }

        init(id: UUID = UUID(), type: String, data: [String:String]) {
            self.id = id
            self.type = type
            self.data = data
        }

        init(from: Decoder) throws {
            let container = try from.container(keyedBy: CodingKeys.self)
            let type = try container.decode(String.self, forKey: .type)
            let data = try container.decode([String:String].self, forKey: .data)
            self.init(type: type, data: data)
        }
    }
    
    let lines: [Line]
}
```