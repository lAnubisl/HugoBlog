---
title: "Rethinking Code Comments in the AI Era"
date: 2024-01-27T21:40:54Z
keywords: "AI, GitHub CoPilot, Comments, Programming, Software development"
description: "In the realm of software development, the utility of comments in code has long been a subject of debate. Traditionally, many have held the belief that high-quality code should speak for itself, rendering comments unnecessary. This viewpoint advocates for self-explanatory code through well-named variables, functions, and classes, and a logical assembly of the code structure. However, the advent of AI-assistants like GitHub Copilot is challenging this notion, ushering in a new perspective on the role of comments in coding. In this blog post, we'll explore how AI is reshaping our approach to commenting, transforming it from a tedious task to an integral part of the coding process."
summary: " ![](/images/i-like-to-write-comments-now/logo.jpeg) In the realm of software development, the utility of comments in code has long been a subject of debate. Traditionally, many have held the belief that high-quality code should speak for itself, rendering comments unnecessary. This viewpoint advocates for self-explanatory code through well-named variables, functions, and classes, and a logical assembly of the code structure. However, the advent of AI-assistants like GitHub Copilot is challenging this notion, ushering in a new perspective on the role of comments in coding. In this blog post, we'll explore how AI is reshaping our approach to commenting, transforming it from a tedious task to an integral part of the coding process."
draft: false
---

![](/images/i-like-to-write-comments-now/logo.jpeg)

## Introduction

In the traditional realm of software development, the necessity and utility of comments within code have often been subjects of debate. The prevailing wisdom has been that exemplary code should essentially speak for itself, reducing the need for comments. This belief is anchored in the principle that clear, self-explanatory code is achieved through meticulously named variables, functions, classes, and a logically structured assembly of code components. However, the emergence of AI-assistants like GitHub Copilot is challenging this long-held notion, bringing a fresh perspective to the role and relevance of comments in coding.

## The Traditional View of Comments

Traditionally, comments in code were often perceived as superfluous. The argument was straightforward: if the code is well-written, it should be inherently understandable without additional commentary. This philosophy encouraged developers to focus on refining their code—prioritizing readability and simplicity over explanatory annotations. In essence, the act of commenting was seen more as a crutch for poorly written code rather than a tool for enhancement.

## Exceptions to the Rule

Despite this, there were acknowledged exceptions to the anti-commentary stance:

- Public Interface Documentation: For code intended for use by others, particularly in libraries and frameworks, comments providing usage hints are invaluable.
- Complex Logic Explanation: In instances of complex logic or non-obvious solutions, such as performance optimizations (like the Fast Inverse Square Root algorithm), comments can offer crucial insights.
- TODOs in Team Settings: Comments indicating future tasks (TODOs) are particularly useful in collaborative environments, aiding in task handovers and work continuity.

## The Impact of AI-Assistants

The emergence of AI-assistants like GitHub Copilot represents a pivotal shift in the role of comments in coding. Far from being mere annotations, comments are evolving into a crucial communication channel between the developer and the AI. This interaction allows the developer to impart contextual information to the AI, which, in turn, leverages this understanding to generate more accurate and relevant code suggestions.

In this new paradigm, comments are no longer static entities; they actively shape the AI's comprehension of the task at hand. As a developer articulates the intent, logic, or specific nuances of the code through comments, the AI assimilates this information, enhancing its ability to provide targeted, context-aware suggestions. This dynamic creates a symbiotic relationship where the quality of AI suggestions is directly influenced by the clarity and detail of the developer's comments.

For instance, when a developer is coding a complex algorithm and uses comments to explain the rationale or specific considerations behind their approach, the AI assistant can use this information to suggest more appropriate code structures or optimizations. This not only streamlines the coding process but also elevates the quality of the resulting code.

Thus, in the era of AI-assisted coding, comments transcend their traditional role of mere code explanation. They become an integral part of the coding dialogue, facilitating a more nuanced and informed interaction between the developer and the AI. This advancement not only addresses the longstanding concerns associated with comments—such as the time spent writing them and their synchronization with the code—but also opens up new avenues for more efficient and effective coding practices.

## An Illustrative Example

Consider the following C# code snippet:

``` csharp
    private static float TopHalfToBottomHalfAngle(bool[,] digitBitMask)
    {
        // An image is rotated around the center. In order to make it straight, we need to find
        // the angle of rotation.

        // split image to 'top' and 'bottom' halfes.
        var middle = digitBitMask.GetLength(1) / 2;

        // allocate variables to store the max and min X coordinates of the black pixels in
        // each half.
        var maxXTop = 0;
        var minXTop = int.MaxValue;
        var maxXBottom = 0;
        var minXBottom = int.MaxValue;

        // find the max and min X coordinates of the black pixels in each half
        for (int y = 0; y < digitBitMask.GetLength(1); y++)
            for (int x = 0; x < digitBitMask.GetLength(0); x++)
                if (digitBitMask[x, y])
                    // if the pixel is black, check if it's in the top or bottom half
                    if (y < middle)
                    {
                        // if it's in the top half, check if it's the max or min X
                        if (x > maxXTop) maxXTop = x;
                        if (x < minXTop) minXTop = x;
                    }
                    else
                    {
                        // if it's in the bottom half, check if it's the max or min X
                        if (x > maxXBottom) maxXBottom = x;
                        if (x < minXBottom) minXBottom = x;
                    }

        // now we have the max and min X coordinates of the black pixels in each half
        // calculate the middle X coordinate of black pixels in each half
        var middleTop = (maxXTop + minXTop) / 2;
        var middleBottom = (maxXBottom + minXBottom) / 2;

        /*  calculate the angle of rotation
            if middleTop == middleBottom, then the angle is 0
            if middleTop > middleBottom, then the angle is negative which means the image is 
                rotated clockwise
            if middleTop < middleBottom, then the angle is positive which means the image is
                rotated counter-clockwise

                       │----(*) middle top
                       │    /   point
                       │ @ /
                       │  /
                       │ /
                       │/
        ───────────────┼───────────────
                      /│
                     / │
                    /  │
         middle    / @ │
         bottom   /    │
         point  (*)----│


            we need to find the angle of the line between the middleTop and middleBottom points
            and the Y axis of the center of the coordinate system placed in the center of the 
            rectangle created by middleTop and middleBottom points
        */
        var middleTopPointCoordinates = 
            new Point(
                middleTop - middleBottom,
                digitBitMask.GetLength(1) / 2);
        var angle = Math.Atan2(middleTopPointCoordinates.Y, middleTopPointCoordinates.X);
        return (float)(angle * 180 / Math.PI) - 90;
    }
```

This example, though not perfect, illustrates how well-placed comments can significantly aid AI tools in understanding the code's context, leading to better AI-generated suggestions. The synergy between the developer and the AI mimics a pair programming scenario, enhancing both efficiency and code quality.

## Addressing Traditional Concerns

With AI-assistants, the two primary concerns about comments are effectively addressed:

- Time Efficiency: AI tools like GitHub Copilot facilitate rapid comment creation, significantly reducing the time traditionally associated with this task.
- Synchronization: The ease of updating comments with AI assistance encourages their regular revision, ensuring they remain in sync with the code.

## Conclusion

The integration of AI-assistants into the coding workflow is a game-changer, reshaping not just our approach to commenting but also our broader interaction with code development. This evolution demands that developers adapt and embrace these tools to stay relevant and efficient. In this new era, comments transcend their traditional role, becoming an active, dynamic participant in the coding process rather than a passive, often-ignored element. The future of coding, with AI as a partner, is not just inevitable—it's already here.