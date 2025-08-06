---
layout: post
title: "Implementing Drag and Drop in UICollectionView and Core Data – Part 2"
date: 2025-07-05 18:30:00 -0400
tags: swift ios core-data
---

# Migration Plan: `PhotoListController` to `UICollectionViewDiffableDataSource`

This document outlines the plan to migrate `PhotoListController` from using `NSFetchedResultsControllerDelegate` and traditional `UICollectionViewDataSource` methods to the more modern `UICollectionViewDiffableDataSource`.

## 1. Overview

The current implementation of `PhotoListController` relies on `NSFetchedResultsController` to manage data from Core Data and its delegate methods to handle UI updates in the `UICollectionView`. This involves manual management of `BlockOperation` objects to batch updates, which can be complex and error-prone.

`UICollectionViewDiffableDataSource` simplifies this process by automatically handling updates to the collection view based on changes in the data. We will replace the existing data source and delegate methods with a diffable data source, which will be updated by the `NSFetchedResultsControllerDelegate`.

## 2. Analysis of Existing Code

- **`PhotoListController.swift`**: This is the main view controller. It sets up the `UICollectionView`, `NSFetchedResultsController`, and handles user interactions.
- **`PhotoListController+NSFetchedResultsControllerDelegate.swift`**: This extension implements the delegate methods for the `NSFetchedResultsController`. It manually creates and manages `BlockOperation`s to update the `UICollectionView` when the data changes. This is the primary area that will be refactored.
- **`PhotoListController+UICollectionViewDataSource.swift`**: This is implicitly implemented in `PhotoListController.swift` with `collectionView(_:numberOfItemsInSection:)` and `collectionView(_:cellForItemAt:)`. These will be replaced by the diffable data source's cell provider.
- **`PhotoListController+UICollectionViewDelegate.swift`**: This is also implicitly implemented in `PhotoListController.swift`. The delegate methods for selection (`didSelectItemAt` and `didDeselectItemAt`) will be kept, but the data source-related delegate methods will be removed.
- **`PhotoListController+DZNEmptyDataSetSource.swift` and `PhotoListController+DZNEmptyDataSetDelegate.swift`**: These extensions handle the display of an empty state for the collection view. This functionality will need to be preserved.
- **`PhotoListController+UIImagePickerViewDelegate.swift`**: This handles image picking and will not be affected.
- **`PhotoListController+UICollectionViewDelegateFlowLayout.swift`**: This handles the layout of the collection view cells and will not be affected.

## 3. Migration Steps

### Step 1: Define Section and Item Identifiers

First, we need to define the section and item identifiers for our diffable data source. Since we have a single section, we can use an enum for the sections. The item identifier will be the `Photo` object itself, which must be hashable.

```swift
// In PhotoListController.swift

enum Section {
    case main
}

// Make Photo hashable
extension Photo: Hashable {
    public static func == (lhs: Photo, rhs: Photo) -> Bool {
        return lhs.objectID == rhs.objectID
    }

    public func hash(into hasher: inout Hasher) {
        hasher.combine(objectID)
    }
}
```

### Step 2: Create the Diffable Data Source

We will create a new property for the `UICollectionViewDiffableDataSource` in `PhotoListController.swift`.

```swift
// In PhotoListController.swift

private var dataSource: UICollectionViewDiffableDataSource<Section, Photo>!
```

### Step 3: Configure the Data Source

In `viewDidLoad`, we will configure the diffable data source. This involves creating the data source and providing a cell provider closure.

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

### Step 4: Update the `NSFetchedResultsControllerDelegate`

The `NSFetchedResultsControllerDelegate` methods will be simplified. Instead of managing `BlockOperation`s, we will create a new snapshot of the data and apply it to the data source in `controllerDidChangeContent`.

```swift
// In PhotoListController+NSFetchedResultsControllerDelegate.swift

func controller(_ controller: NSFetchedResultsController<NSFetchRequestResult>, didChangeContentWith snapshot: NSDiffableDataSourceSnapshotReference) {
    guard let dataSource = self.dataSource else { return }

    var newSnapshot = NSDiffableDataSourceSnapshot<Section, Photo>()
    newSnapshot.appendSections([.main])
    newSnapshot.appendItems(controller.fetchedObjects as? [Photo] ?? [])

    dataSource.apply(newSnapshot, animatingDifferences: true)

    // Refresh empty state
    self.collectionView?.reloadEmptyDataSet()

    // Refresh edit button
    self.editButtonItem.isEnabled = controller.fetchedObjects?.count != 0

    // Check if there is nothing else let to display
    if (controller.fetchedObjects == nil || controller.fetchedObjects?.count == 0) {
        // Exit editing mode
        self.setEditing(false, animated: true)
    }
}
```

We will remove `controllerWillChangeContent`, `controller(_:didChange:at:for:newIndexPath:)`, and `controller(_:didChangeSection:atSectionIndex:for:)`. The new `controller(_:didChangeContentWith:)` will handle all updates.

### Step 5: Remove Old Data Source Methods

The following methods in `PhotoListController.swift` will be removed:

- `collectionView(_:numberOfItemsInSection:)`
- `collectionView(_:cellForItemAt:)`

### Step 6: Initial Data Load

In `viewDidLoad`, after configuring the data source, we will perform the initial fetch and apply the snapshot.

```swift
// In PhotoListController.swift, inside viewDidLoad()

// Fetch data
do {
    try self.fetchedResultsController.performFetch()
    updateDataSource()
    self.editButtonItem.isEnabled = fetchedResultsController.fetchedObjects?.count != 0
} catch {
    fatalError("Error fetching guns: \(error)")
}

// new helper function
private func updateDataSource() {
    var snapshot = NSDiffableDataSourceSnapshot<Section, Photo>()
    snapshot.appendSections([.main])
    snapshot.appendItems(fetchedResultsController.fetchedObjects ?? [])
    dataSource.apply(snapshot, animatingDifferences: false)
}
```

### Step 7: Update `setEditing`

The `setEditing` method reloads cells, which is no longer necessary in the same way. We can simplify it.

```swift
// In PhotoListController.swift

override func setEditing(_ editing: Bool, animated: Bool) {
    super.setEditing(editing, animated: animated)

    // Toggle multiple selection option
    self.collectionView?.allowsMultipleSelectionDuringEditing = editing

    // Update navigation item title
    self.navigationItem.title = editing ? "Select photos" : Strings.TITLE_PHOTOS

    // Update edit button title
    self.editButtonItem.title = editing ? Strings.ACTION_DONE : Strings.ACTION_SELECT

    // Update camera button state
    self.allowCamera(flag: !editing)

    // Update action and trash button state
    self.allowActionsOnSelection(flag: false)
}
```

## 4. Testing and Verification

After the migration, the following functionality should be tested to ensure it works as expected:

- **Adding a new photo**: The collection view should update correctly.
- **Deleting a photo**: The collection view should update correctly.
- **Selecting and deselecting photos in editing mode**: The UI should update correctly.
- **Empty state**: The empty state message should be displayed when there are no photos.
- **Rotation**: The layout should update correctly on device rotation.

This migration will result in cleaner, more maintainable code that leverages modern iOS development practices.
