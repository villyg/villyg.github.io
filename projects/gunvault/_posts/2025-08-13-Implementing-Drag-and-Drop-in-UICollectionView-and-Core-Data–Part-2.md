---
layout: post
title: "Implementing Drag and Drop in UICollectionView and Core Data – Part 2"
date: 2025-08-12 18:30:00 -0400
tags: swift ios core-data
---

In [Part 1]({% link projects/gunvault/_posts/2025-08-04-Implementing-Drag-and-Drop-in-UICollectionView-and-Core-Data–Part-1.md %}) of this series, I walked through the process of adding drag-and-drop functionality to a `UICollectionView` backed by Core Data. While I got it working, I mentioned that the solution, while functional, wasn't as modern as it could be. The `NSFetchedResultsControllerDelegate` implementation, with its manual block operations, felt a bit clunky and error-prone. This post is about that additional refactoring: moving from the traditional `NSFetchedResultsControllerDelegate` to the more modern `UICollectionViewDiffableDataSource`.

- [Background](#background)
- [The Plan: A Modern Approach](#the-plan-a-modern-approach)
  - [Step 1: Laying the Foundation](#step-1-laying-the-foundation)
  - [Step 2: Creating the Diffable Data Source](#step-2-creating-the-diffable-data-source)
  - [Step 3: Tying it to Core Data](#step-3-tying-it-to-core-data)
  - [Step 4: The Initial Load](#step-4-the-initial-load)
- [The Result: Cleaner, Simpler Code](#the-result-cleaner-simpler-code)
- [Conclusion](#conclusion)

## Background

To recap, the `PhotoListController` was using a `NSFetchedResultsController` to manage photos from Core Data. The delegate methods were responsible for updating the `UICollectionView` in response to data changes. This involved a lot of manual work with `BlockOperation` to batch updates, which, as I discovered in [Part 1]({% link projects/gunvault/_posts/2025-08-04-Implementing-Drag-and-Drop-in-UICollectionView-and-Core-Data–Part-1.md %}), can be tricky to get right.

The goal of this refactoring was to simplify this process by leveraging `UICollectionViewDiffableDataSource`, which Apple introduced in iOS 13. Diffable data sources automatically handle the complexities of updating the UI, which means less code, fewer bugs, and a happier developer (me).

## The Plan: A Modern Approach

The plan was to replace the existing `UICollectionViewDataSource` and `NSFetchedResultsControllerDelegate` methods with a `UICollectionViewDiffableDataSource`. Here’s how I broke it down.

### Step 1: Laying the Foundation

First, I needed to define the section and item identifiers for the diffable data source. Since my collection view only has one section, this was straightforward. The item identifier would be the `Photo` object itself, because the `NSManagedObject` it inherits from already conforms to `Hashable`.

```swift
// In PhotoListController.swift

enum Section {
    case main
}

```

### Step 2: Creating the Diffable Data Source

Next, I created a new property for the `UICollectionViewDiffableDataSource` in `PhotoListController.swift`.

```swift
// In PhotoListController.swift

private var dataSource: UICollectionViewDiffableDataSource<Section, Photo>!
```

Then, in `viewDidLoad`, I configured the data source. This involved creating the data source and providing a "cell provider" closure, which is responsible for configuring and returning a cell for a given `IndexPath` and `Photo`.

```swift
// In PhotoListController.swift, inside viewDidLoad()

dataSource = UICollectionViewDiffableDataSource<Section, Photo>(collectionView: collectionView!) {
    (collectionView: UICollectionView, indexPath: IndexPath, photo: Photo) -> UICollectionViewCell? in
    let cell = collectionView.dequeueReusableCell(withReuseIdentifier: PhotoCell.reuseIdentifer, for: indexPath) as! PhotoCell
    if let data = photo.thumbnailImage?.imageData as Data?, let image = UIImage(data: data) {
        cell.imageView.image = image
    }
    return cell
}
```

This code replaces the old `collectionView(_:cellForItemAt:)` method.

### Step 3: Tying it to Core Data

This is where the magic happens. The `NSFetchedResultsControllerDelegate` methods were significantly simplified. Instead of manually managing `BlockOperation`s, I just needed to create a new snapshot of the data and apply it to the data source in the `controller(_:didChangeContentWith:)` method.

```swift
// In PhotoListController+NSFetchedResultsControllerDelegate.swift

func controller(_ controller: NSFetchedResultsController<NSFetchRequestResult>, didChangeContentWith snapshot: NSDiffableDataSourceSnapshotReference) {
    guard let dataSource = self.dataSource else { return }

    var newSnapshot = NSDiffableDataSourceSnapshot<Section, Photo>()
    newSnapshot.appendSections([.main])
    newSnapshot.appendItems(controller.fetchedObjects as? [Photo] ?? [])

    dataSource.apply(newSnapshot, animatingDifferences: true)

    // Refresh empty state and other UI elements
    self.collectionView?.reloadEmptyDataSet()
    self.editButtonItem.isEnabled = controller.fetchedObjects?.count != 0
    if (controller.fetchedObjects == nil || controller.fetchedObjects?.count == 0) {
        self.setEditing(false, animated: true)
    }
}
```

With this change, I was able to remove the old `controllerWillChangeContent`, `controller(_:didChange:at:for:newIndexPath:)`, and `controllerDidChangeContent` methods. A huge simplification!

### Step 4: The Initial Load

Finally, I updated the initial data load in `viewDidLoad`. After performing the initial fetch, I just needed to call a helper function to update the data source with the fetched data.

```swift
// In PhotoListController.swift, inside viewDidLoad()

// Fetch data
do {
    try self.fetchedResultsController.performFetch()
    updateDataSource()
    self.editButtonItem.isEnabled = fetchedResultsController.fetchedObjects?.count != 0
} catch {
    fatalError("Error fetching photos: \(error)")
}

// new helper function
private func updateDataSource() {
    var snapshot = NSDiffableDataSourceSnapshot<Section, Photo>()
    snapshot.appendSections([.main])
    snapshot.appendItems(fetchedResultsController.fetchedObjects ?? [])
    dataSource.apply(snapshot, animatingDifferences: false)
}
```

## The Result: Cleaner, Simpler Code

After the refactoring, the `PhotoListController` is much cleaner and easier to understand. The `NSFetchedResultsControllerDelegate` extension is now tiny, and I was able to delete a lot of the old, complex code for managing collection view updates.

The drag-and-drop functionality still works perfectly, but now it's built on a much more robust and modern foundation.

## Conclusion

This refactoring was a great investment. It not only modernized the codebase but also made it more resilient to bugs and helped me eliminate a ton of code and unit tests. If you're still using the old `NSFetchedResultsControllerDelegate` with `UICollectionView`, I highly recommend looking into `UICollectionViewDiffableDataSource`. It's a game-changer.

I hope this two-part series has been helpful. If you have any questions or feedback, feel free to reach out!