---
title: "Attimage"
linkTitle: "Attimage"
description: "Generate an attachment image asset definition for rendering features."
weight: 70
---

An attachment image asset definition used by rendering features -- render targets, depth buffers, and other intermediate rendering surfaces. Unlike the component templates, this doesn't generate any C++ source; it produces a single data file with your chosen dimensions and pixel format already filled in.

{{< image-width src="/images/user-guide/editor/class-wizard/templates/attimage-template-screen.png" width="450" alt="The wizard with Attimage selected" >}}

**What you'll fill in:**

| Field | What it does |
|---|---|
| Width / Height | The image's dimensions in pixels. Default `1920` x `1080`. |
| Format | The pixel format for the image, chosen from a dropdown of engine-supported GPU formats. Default `R8G8B8A8_UNORM`. |
| Is Unique | Marks the image as unique when it's generated. |

Fill in a **Name** and pick the **Gem** you're adding it to, then select **Create**.

For the full `template.json` schema, see [Attimage](/docs/engine-dev/tools/class-wizard/templates/attimage/) in the Developer Guide.
