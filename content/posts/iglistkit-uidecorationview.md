---
title: "Decorating UICollectionViewCells"
description: "UICollectionViewCells, with shadows!"
author: "njooma"
date: 2018-10-04T17:46:10-04:00
tags: [UICollectionView, IGListKit, Decoration View, UICollectionReusableView]
---

`UICollectionViewCell`s, with more *\*\~\*pIzAzZ\*\~\**
<!--more-->

#### WORKING CODE AT THE BOTTOM OF THE POST ####

One day, while browsing Twitter at work, I saw that Jesse Squires (of JSQMessageViewController fame) retweeted that there was an update to [IGListKit](https://github.com/instagram/IGListKit)

{{< tweet 961298693742911491 >}}

Interested, I dove in and realized that IGListKit is not only a really powerful set of APIs on top of `UICollectionView`, it would probably make my life easier trying to create a view to spec. (Browsing Twitter at work for work, nothing wrong there...) This isn't a post on the amazing features and life-changing nature of IGListKit. Rather, this is about a specific problem I faced that took me an embarrassingly long time to get to a reasonable solution. 

But to understand the problem, you first have to understand the basics of IGListKit. And the basics are: you group a series of collection view cells into a section, and treat that section as a single unit with its own controller. You can also group sections together into a super-section and treat that as a unit. For example, the Instagram feed: You could treat the image, the actions, and the comment sections as individual sections acting as a unit. 

So what I needed was something with 5 components, expanding to a maximum of 9 components. Nothing too fancy, just a title, description, image, caption, and a view more button. When the view more button is pressed, a detailed description and up to 3 more components expand in. Pretty straightforward:

![Section layout](/static/iglistkit-uidecorationview/image1.png)

But of course, designers can't ever just let it simple. This is what they actually wanted:

![Section design](/static/iglistkit-uidecorationview/image2.png)

Notice the corner radius and drop shadow? Actually that's not too bad. Pretty tame. I didn't figure it would take me too long to bring it up to design. 

### I was mistaken ###

What IGListKit promotes, and what I had done, was made each of the components (title, description, etc.) into it's own cell, and then had the `SectionController` combine them into a section. So far so good. Now let's try to style these sections.

#### Corner Radius ####
This one's pretty simple: go into each individual `UICollectionViewCell` that makes up the section, grab the `layer` and give it a `cornerRadius`.

![Corner radius fail](/static/iglistkit-uidecorationview/image3.png)

Oh... Ok not insurmountable, just tell the `SectionController` to move each cell up by the `cornerRadius` amount! It's hacky, but no one will know.

#### Drop Shadow ####

So with the corner radius taken care of, it was time to tackle the drop shadow. So easy! Just give each cell a drop shadow! Since I moved each cell up already, each successive cell should block the shadow of the cell above it. So for the first component, the top and sides of the shadow are visible; for the last component we'll have the sides and bottom; and for everything in between only, the sides will show. Perfect! Wow my hack from above totally saved me so much time!

And honestly, much to my surprise, it kinda worked. 

![Drop shadows: easy peasy](/static/iglistkit-uidecorationview/image4.png)

Wow this is awesome! Let me scroll and see my beautifu--uuuuuuck. 

![Drop shadows: beautifuck](/static/iglistkit-uidecorationview/image5.png)

Some of you might have seen this coming. But for an explanation: `UICollectionViews` automatically recycle cells as they are scrolled off screen. Which means cells aren't always placed in hierarchical order. While this was fine for my `cornerRadius` hack since all the backgrounds were the same, this doesn't work for shadows when I need each cell to be placed on top of the previous cell.

Not wanting to create more hacks or lose the performance benefits of recycling cells, I decided to query the Internet on how to set up a drop shadow for a collection view section using IGListKit. And this is what I found this [GitHub Issue](https://github.com/Instagram/IGListKit/issues/805) that was exactly what I was asking! And the answer was right there! Use "decoration view API" instead!

... what are "decoration views"

## Decoration Views ##

[From Apple](https://developer.apple.com/library/archive/documentation/WindowsViews/Conceptual/CollectionViewPGforIOS/CreatingCustomLayouts/CreatingCustomLayouts.html#//apple_ref/doc/uid/TP40012334-CH5-SW12):

> Decoration views are visual adornments that enhance the appearance of your collection view layouts. Unlike cells and supplementary views, decoration views provide visual content only and are thus independent of the data source. You can use them to provide custom backgrounds, fill in the spaces around cells, or even obscure cells if you want. Decoration views are defined and managed solely by the layout object and do not interact with the collection view’s data source object. 

Sounds exactly like what I need. Using a few tutorials and [StackOverflow answers](https://stackoverflow.com/questions/12810628/uicollectionview-decoration-view/39218108#39218108), I eventually got to this:

{{< highlight swift "linenos=inline,nowrap=true" >}}
class ShadowDecorationCollectionReusableView: UICollectionReusableView {
    
    override init(frame: CGRect) {
        super.init(frame:frame)
        
        // Add shadow
        backgroundColor = UIColor.gray
        layer.cornerRadius = 4
        layer.shadowColor = UIColor.black.cgColor
        layer.shadowOpacity = 0.2
        layer.shadowRadius = 4
        layer.shadowOffset = CGSize(width: 0, height: 4)
    }
    
    required init?(coder aDecoder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
        
}


class ShadowFlowLayout: UICollectionViewFlowLayout {

    override init() {
        super.init()
        
        // Register the decoration view cell
        self.register(ShadowDecorationCollectionReusableView.self, forDecorationViewOfKind: "ShadowDecoration")
    }
    
    required init?(coder aDecoder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    override func layoutAttributesForDecorationView(ofKind elementKind: String, at indexPath: IndexPath) -> UICollectionViewLayoutAttributes? {
        // Get the attributes for the cell at the current index path
        if let cellAttrs = self.layoutAttributesForItem(at: indexPath) {
            // Get the attributes for the decoration view cell
            let attrs = UICollectionViewLayoutAttributes(forDecorationViewOfKind: elementKind, with: indexPath)
            // Give the decoration view the same frame as the content cell
            attrs.frame = cellFrame.frame
            // Move the decoration cell to the back
            attrs.zIndex = -1
            return attrs
        }
        return nil
    }
    
    override func layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]? {
        var attributes = super.layoutAttributesForElements(in: rect)!
        // For each cell that's currently visible...
        for attribute in attributes {
            // Get the decoration attributes and add that to the list of attributes
            if let decAttrs = layoutAttributesForDecorationView(ofKind: "ShadowDecoration", at: attribute.indexPath) {
                attributes.append(decAttrs)
            }
        }
        return attributes
    }
    
}
{{< / highlight >}}

This basically does the exact same thing I was doing earlier: every cell is partnered with a decoration view with a shadow. But this time, I'm forcing the shadow to always be behind everything with `attrs.zIndex = -1`. Looking good and it works with scroll. 

And it did work. For a few months. 

I ran into issues with sections containing cells of different widths: now the sides of the shadows don't line up. My shortcuts have failed and I need to solve the root problem now: how to add a drop shadow to an entire section using decoration views.

More searches later[^1][^2][^3] and I had an idea. I was being pointed in the right direction and almost had everything I needed. The only difference was that I was treating a section as a single unit, while all these online articles were treating them as what they're traditionally used as: groupings of individual cells. But the suggestions were all the same:

From the `indexPath`, get the `section`. From the `section`, get the first cell and the last cell. Finally, create a decoration view whose frame spans from the origin of the first cell to the `maxX` and `maxY` of the last cell.

{{< highlight swift >}}
class SectionShadowFlowLayout: UICollectionViewFlowLayout {
    
    override init() {
        super.init()
        
        self.register(SectionShadowCollectionReusableView.self, forDecorationViewOfKind: "SectionShadowCollectionReusableView")
    }
    
    required init?(coder aDecoder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    override func layoutAttributesForDecorationView(ofKind elementKind: String, at indexPath: IndexPath) -> UICollectionViewLayoutAttributes? {
        
        // Get the first and last attributes in the section
        guard let itemCount = self.collectionView?.numberOfItems(inSection: indexPath.section),
            let first = self.layoutAttributesForItem(at: IndexPath(item: 0, section: indexPath.section)),
            let last = self.layoutAttributesForItem(at: IndexPath(item: itemCount-1, section: indexPath.section)) else {
                return nil
        }
        
        // Get the frame for the decoration view
        let origin = first.frame.origin
        let width = first.frame.width
        let height = last.frame.maxY-first.frame.minY
        let size = CGSize(width: width, height: height)
        
        // Do the same as before
        let attrs = UICollectionViewLayoutAttributes(forDecorationViewOfKind: elementKind, with: indexPath)
        attrs.frame = CGRect(origin: origin, size: size)
        attrs.zIndex = -1
        decorationAttributesForSection[indexPath.section] = attrs
        return attrs
    }
    
    override func layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]? {
        var attributes = super.layoutAttributesForElements(in: rect)!
        
        for attribute in attributes {
            let decAttrs = layoutAttributesForDecorationView(ofKind: "SectionShadowCollectionReusableView", at: attribute.indexPath) {
            attributes.append(decAttrs)
            sections.insert(attribute.indexPath.section)
        }
        return attributes
    }
    
}
{{< / highlight >}}

If you try to run this code, you'll notice a crippling defect: it's incredibly slow, and you receive multiple shadows for each section. This is because as the user scrolls, I'm calculating the shadow of the entire section *for each visible cell in the section*. So if I have 5 visible cells, I'm doing this calculation 5 times for the same section, and adding 5 decoration views. To get around this, I started saving which sections I had already checked.

{{< highlight swift >}}
override func layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]? {
    var attributes = super.layoutAttributesForElements(in: rect)!
    
    // Make sure we don't run the calculations multiple times for the same section
    var sections = Set<Int>()
    for attribute in attributes {
        if !(sections.contains(attribute.indexPath.section)),
            let decAttrs = layoutAttributesForDecorationView(ofKind: "SectionShadowCollectionReusableView", at: attribute.indexPath) {
            attributes.append(decAttrs)
            sections.insert(attribute.indexPath.section)
        }
    }
    return attributes
}
{{< / highlight >}}

So while this sped things up, it didn't solve the multiple decoration views. That took me a while to track down, but it's because once decoration views are added, they aren't removed (even though we're adding the attributes every single time). So I introduced a dictionary that stored sections with their attributes, and voilà! 

## Working Code ##

Working code to add shadows to entire sections in a UICollectionView:

{{< highlight swift >}}
class SectionShadowFlowLayout: UICollectionViewFlowLayout {
    
    private var decorationAttributesForSection = [Int: UICollectionViewLayoutAttributes]()
    
    override init() {
        super.init()
        
        self.register(SectionShadowCollectionReusableView.self, forDecorationViewOfKind: "SectionShadowCollectionReusableView")
    }
    
    required init?(coder aDecoder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    override func layoutAttributesForDecorationView(ofKind elementKind: String, at indexPath: IndexPath) -> UICollectionViewLayoutAttributes? {
        
        guard let lastIndex = self.collectionView?.numberOfItems(inSection: indexPath.section),
            let first = self.layoutAttributesForItem(at: IndexPath(item: 0, section: indexPath.section)),
            let last = self.layoutAttributesForItem(at: IndexPath(item: lastIndex-1, section: indexPath.section)) else {
                return nil
        }
        
        let origin = first.frame.origin
        let width = first.frame.width
        let height = last.frame.maxY-first.frame.minY
        let size = CGSize(width: width, height: height)
        
        // Have to return the same attribute object from cache or else it'll get added twice
        let attrs = decorationAttributesForSection[indexPath.section] ?? UICollectionViewLayoutAttributes(forDecorationViewOfKind: elementKind, with: indexPath)
        attrs.frame = CGRect(origin: origin, size: size)
        attrs.zIndex = -1
        decorationAttributesForSection[indexPath.section] = attrs
        return attrs
    }
    
    override func layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]? {
        var attributes = super.layoutAttributesForElements(in: rect)!
        
        // Make sure we don't run the calculations multiple times for the same section
        var sections = Set<Int>()
        for attribute in attributes {
            if !(sections.contains(attribute.indexPath.section)),
                let decAttrs = layoutAttributesForDecorationView(ofKind: "SectionShadowCollectionReusableView", at: attribute.indexPath) {
                attributes.append(decAttrs)
                sections.insert(attribute.indexPath.section)
            }
        }
        return attributes
    }
    
}
{{< / highlight >}}

[^1]: https://mpospese.com/2012/12/11/decorationviews/
[^2]: http://strawberrycode.com/blog/how-to-create-a-section-background-in-a-uicollectionview-in-ios-swift/
[^3]: https://stackoverflow.com/questions/13609204/how-to-change-background-color-of-a-whole-section-in-uicollectionview
