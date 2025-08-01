---
layout: post
title: "Auto-sizing UIImage on top of UITableView during rotation"
date: 2024-12-27 17:37:16 -0400
tags: [swift, ios]
permalink: /posts/:title
---

- [Introduction](#introduction)
- [Set Up](#set-up)
- [Primary image](#primary-image)
- [Rotation support](#rotation-support)

## Introduction

As part of version 1.2 of Gun Vault I decided to include the functionality of having the primary image of a gun included on top of the Gun Details screen. I wanted the image to take 1/3 of the screen when the device is in portrait mode …

![Portrait](/images/post-2017-12-27/snapshot-portrait.png)


and full screen when in landscape mode.

![Full screen](/images/post-2017-12-27/snapshot-landscape.png)

I thought this would be a simple task until I ended up spending over 2 hrs trying to fine-tune it so I figured I’d write about it.

## Set up

We’ll keep it very simple - 1 grouped-style table with 3 sections and 5 rows inside each section.

```swift
import UIKit
class ViewController: UITableViewController {
    
    convenience init() {
        
        self.init(style: UITableViewStyle.grouped)
        
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
        self.navigationItem.title = "Gun details"
        
    }
    
    
    // ***********************************************
    // MARK: - UITableViewDataSource
    // ***********************************************
    
    
    
    override func numberOfSections(in tableView: UITableView) -> Int {
        
        return 3
        
    }
    
    
    override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        
        return 5
        
    }
    
    
    override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        
        // since it is a static view we don't care about recycling at this time
        let result: UITableViewCell = UITableViewCell(style: .value1, reuseIdentifier: "cell")
        
        result.textLabel?.text = "Text for row at: \(indexPath.row)"
        result.detailTextLabel?.text = "Details for row at: \(indexPath.row)"
        
        return result
        
    }
    
}
```

## Primary image

In order to add a primary image on top we need to override 2 methods: `heightForHeaderInSection` and `viewForHeaderInSection`. The first one controlls the height of placeholder we are going to use for the image and the second one provides the actual image to be displayed.

Note that we are not using the device orientation but the statusBar orientation instead. This is because while the device can be rotated - the interface could be locked for rotation in the same time.

```swift
override func tableView(_ tableView: UITableView, heightForHeaderInSection section: Int) -> CGFloat {
    
    if section == 0 {
        let orientation = UIApplication.shared.statusBarOrientation
        
        if orientation.isLandscape {
            return view.frame.height
        } else {
            return view.frame.height / 3
        }
    }
    
    return UITableViewAutomaticDimension
    
}
```

```swift

override func tableView(_ tableView: UITableView, viewForHeaderInSection section: Int) -> UIView? {
 
    // We'll assume that there is only one section for now.
    
    if section == 0 {
        
        let imageView: UIImageView = UIImageView()
        imageView.clipsToBounds = true
        imageView.contentMode = .scaleAspectFill
        imageView.image =  UIImage(named: "gun")!
        return imageView
    }
    
    return nil
    
}

```

## Rotation support

The last thing that is left to add it the rotation support. Note that there are 3 phases that allow hooks. For our purpose we’ll use the before rotation and during rotation to invalidate the table layout. My first try was actually adding `tableView.reloadData()` instead but it resulted in a flicker of the screen so `beginUpdates()` and `endUpdates()` ended up being the better choice.

```swift
override func viewWillTransition(to size: CGSize, with coordinator: UIViewControllerTransitionCoordinator) {
    super.viewWillTransition(to: size, with: coordinator)
    
    //        print("will execute before rotation")
    
    self.tableView.beginUpdates()
    
    coordinator.animate(alongsideTransition: { (context: UIViewControllerTransitionCoordinatorContext) in
        
        //            print("will execute during rotation")
        
        self.tableView.endUpdates()
        
    }) { (context: UIViewControllerTransitionCoordinatorContext) in
        
        //            print("will execute after rotation")
        
    }
    
}
```

That’s it. Now you have a nice table view with a smart primary image.

For the full codebase, check out the [GitHub repository](https://github.com/villyg/AutoSizingSectionViewImage).