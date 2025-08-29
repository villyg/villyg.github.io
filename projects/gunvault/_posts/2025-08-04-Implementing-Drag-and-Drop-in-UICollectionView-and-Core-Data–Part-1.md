---
layout: post
title: "Implementing Drag and Drop in UICollectionView and Core Data – Part 1"
date: 2025-08-04 18:30:00 -0400
tags: swift ios core-data
featured_image: UIKit.png
---

One of the feature requests I received was to allow users to rearrange gun photos in a custom order. Since I was planning other changes to the photos module, I decided to tackle this request first as it would lay a solid foundation for future updates. What seemed like a simple task turned out to be a bit more complex than expected, so I decided to blog about it.
<!--more-->

- [Background](#background)
- [Plan](#plan)
  - [Step 1: Updating the model](#step-1-updating-the-model)
  - [Step 2: Adding Drag-and-Drop Support](#step-2-adding-drag-and-drop-support)
  - [Step 3: Assigning `displayOrder` to New Photos](#step-3-assigning-displayorder-to-new-photos)
- [Problem](#problem)
- [Root cause analysis](#root-cause-analysis)
- [Solution](#solution)
- [Conclusion](#conclusion)
<!-- more -->
## Background

Since this is a Core Data-enabled application, the `PhotoListController` is a `UICollectionViewController` and also acts as a delegate for an `NSFetchedResultsController`. It retrieves photos and displays them in a 3-column grid using `UICollectionViewDelegateFlowLayout`. By default, photos are sorted by the date/time they were captured or imported. The controller also handles capture/import, selection, deletion, and exporting, all of which needed to be considered.

The controller’s structure is typical:

```swift
class PhotoListController: UICollectionViewController {
    lazy var fetchedResultsController: NSFetchedResultsController<Photo> = {
        let predicate = NSPredicate(format: "gun == %@", self.gun)
        let sortDescriptor = NSSortDescriptor(key: "createdOn", ascending: true)
        let fReq: NSFetchRequest<Photo> = Photo.fetchRequest()
        fReq.sortDescriptors = [sortDescriptor]
        fReq.predicate = predicate
        let frc = NSFetchedResultsController(fetchRequest: fReq, managedObjectContext: self.managedObjectContext, sectionNameKeyPath: nil, cacheName: nil)
        return frc
    }()
    // More code here...
}
```

The `NSFetchedResultsControllerDelegate` implementation uses block operations and branching logic for different update types:

```swift
extension PhotoListController: NSFetchedResultsControllerDelegate {
    func controllerWillChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        self.blockOperations.removeAll(keepingCapacity: false)
    }
    // ...section change handling...
    func controller(_ controller: NSFetchedResultsController<NSFetchRequestResult>, didChange anObject: Any, at indexPath: IndexPath?, for type: NSFetchedResultsChangeType, newIndexPath: IndexPath?) {
        // ...insert, update, move, delete logic...
    }
    func controllerDidChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        collectionView!.performBatchUpdates({
            for operation in self.blockOperations {
                operation.start()
            }
        }, completion: { _ in
            self.blockOperations.removeAll(keepingCapacity: false)
            // Additional logic
        })
    }
}
```

The model is straightforward, with UUID support from the start:

```swift
@objc(Photo)
class Photo: NSManagedObject {}

extension Photo {
    @nonobjc class func fetchRequest() -> NSFetchRequest<Photo> {
        return NSFetchRequest<Photo>(entityName: "Photo")
    }
    override func awakeFromInsert() {
        super.awakeFromInsert()
        uniqueId = NSUUID().uuidString
        createdOn = NSDate()
    }
    @NSManaged var uniqueId: String?
    @NSManaged var createdOn: NSDate?
    @NSManaged var gun: Gun?
    @NSManaged var thumbnailImage: PhotoImageThumbnail?
    @NSManaged var fullSizeImage: PhotoImageFullSize?
}
```

## Plan

My plan was to add a custom order property to the model, enable drag-and-drop in the `UICollectionViewController`, and persist the new order in Core Data. Any changes would be broadcast using the existing plumbing between Core Data and the various `NSFetchedResultsController` delegates.

### Step 1: Updating the model

To persist custom order, I introduced a new attribute `displayOrder`:

1. **Update the model version:** I generated a new model version for seamless migration.
2. **Update the Photo entity:** Added a new attribute `displayOrder` (type: Integer 16) to the `Photo` entity.
3. **Update the Swift class:** Added the `displayOrder` property to the Swift class.

```swift
extension Photo {
    @nonobjc class func fetchRequest() -> NSFetchRequest<Photo> {
        return NSFetchRequest<Photo>(entityName: "Photo")
    }
    override func awakeFromInsert() {
        super.awakeFromInsert()
        uniqueId = NSUUID().uuidString
        createdOn = NSDate()
    }
    @NSManaged var uniqueId: String?
    @NSManaged var createdOn: NSDate?
    @NSManaged var displayOrder: Int16 // New attribute
    @NSManaged var gun: Gun?
    @NSManaged var thumbnailImage: PhotoImageThumbnail?
    @NSManaged var fullSizeImage: PhotoImageFullSize?
}
```

### Step 2: Adding Drag-and-Drop Support

To enable reordering, I modified the controller to implement `UICollectionViewDragDelegate` and `UICollectionViewDropDelegate`:

**Enable Drag and Drop:** In `viewDidLoad`, enable drag interactions and set delegates.

```swift
override func viewDidLoad() {
    super.viewDidLoad()
    // ...existing setup...
    collectionView.dragInteractionEnabled = true
    collectionView.dragDelegate = self
    collectionView.dropDelegate = self
}
```

**Update Fetch Request:** Sort photos by `displayOrder`.

```swift
lazy var fetchedResultsController: NSFetchedResultsController<Photo> = {
    let fetchRequest: NSFetchRequest<Photo> = Photo.fetchRequest()
    fetchRequest.predicate = NSPredicate(format: "gun = %@", self.gun)
    let sortDescriptor = NSSortDescriptor(key: "displayOrder", ascending: true)
    fetchRequest.sortDescriptors = [sortDescriptor]
    // ...rest of initialization...
    return frc
}()
```

**Implement Drag Delegate:** Specify draggable items.

```swift
extension PhotoListController: UICollectionViewDragDelegate {
    func collectionView(_ collectionView: UICollectionView, itemsForBeginning session: UIDragSession, at indexPath: IndexPath) -> [UIDragItem] {
        let photo = fetchedResultsController.object(at: indexPath)
        let itemProvider = NSItemProvider(object: photo.uniqueId! as NSString)
        let dragItem = UIDragItem(itemProvider: itemProvider)
        dragItem.localObject = photo
        return [dragItem]
    }
}
```

**Implement Drop Delegate:** Handle drop and persist new order.

```swift
extension PhotoListController: UICollectionViewDropDelegate {
    func collectionView(_ collectionView: UICollectionView, canHandle session: UIDropSession) -> Bool {
        return session.canLoadObjects(ofClass: NSString.self)
    }
    func collectionView(_ collectionView: UICollectionView, dropSessionDidUpdate session: UIDropSession, withDestinationIndexPath destinationIndexPath: IndexPath?) -> UICollectionViewDropProposal {
        if collectionView.hasActiveDrag {
            return UICollectionViewDropProposal(operation: .move, intent: .insertAtDestinationIndexPath)
        }
        return UICollectionViewDropProposal(operation: .forbidden)
    }
    func collectionView(_ collectionView: UICollectionView, performDropWith coordinator: UICollectionViewDropCoordinator) {
        guard let destinationIndexPath = coordinator.destinationIndexPath,
              let sourceIndexPath = coordinator.items.first?.sourceIndexPath else { return }
        guard var photos = fetchedResultsController.fetchedObjects else { return }
        let photoToMove = photos.remove(at: sourceIndexPath.item)
        photos.insert(photoToMove, at: destinationIndexPath.item)
        for (index, photo) in photos.enumerated() {
            photo.displayOrder = Int16(index)
        }
        do {
            try self.managedObjectContext.save()
        } catch {
            print("Failed to save photo order: \(error)")
        }
    }
}
```

### Step 3: Assigning `displayOrder` to New Photos

I updated the capture/import logic to assign `displayOrder` automatically:

```swift
func imagePickerController(_ picker: UIImagePickerController, didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey : Any]) {
    // ...existing code to get image...
    let photo = Photo(context: coreDataStack.managedContext)
    photo.uuid = UUID()
    photo.createdAt = Date()
    photo.gun = self.gun
    // Assign the next display order
    let currentCount = fetchedResultsController.fetchedObjects?.count ?? 0
    photo.displayOrder = Int16(currentCount)
    // ...rest of the method to set images and save...
    dismiss(animated: true)
}
```

## Problem

After implementing drag-and-drop, the app crashed with the dreaded `attempt to perform an insert and a move to the same index path` error.

## Root Cause Analysis

The issue stemmed from how `NSFetchedResultsControllerDelegate` interprets changes. If a `move` is treated as a `delete` and an `insert`, and the `insert` targets the same index path as another independent insert, a conflict occurs.

## Solution

I modified the logic to ensure that inserts, deletes, and move targets are distinct within a batch update. Instead of using the combined `move` operation, I replaced it with a `delete` at the old index path and an `insert` at the new index path:

```swift
extension PhotoListController: NSFetchedResultsControllerDelegate {
    // ...other logic...
    func controller(_ controller: NSFetchedResultsController<NSFetchRequestResult>, didChange anObject: Any, at indexPath: IndexPath?, for type: NSFetchedResultsChangeType, newIndexPath: IndexPath?) {
        if type == .insert {
            blockOperations.append(BlockOperation { [weak self] in
                self?.collectionView?.insertItems(at: [newIndexPath!])
            })
        } else if type == .update {
            blockOperations.append(BlockOperation { [weak self] in
                self?.collectionView?.reloadItems(at: [indexPath!])
            })
        } else if type == .move {
            blockOperations.append(BlockOperation { [weak self] in
                guard let indexPath = indexPath, let newIndexPath = newIndexPath else { return }
                self?.collectionView?.deleteItems(at: [indexPath])
                self?.collectionView?.insertItems(at: [newIndexPath])
            })
        } else if type == .delete {
            blockOperations.append(BlockOperation { [weak self] in
                self?.collectionView?.deleteItems(at: [indexPath!])
            })
        }
    }
}
```

## Conclusion

With these changes, I had a shippable version. Despite a lengthy debugging session, the refactor was fairly straightforward. The original code, written in 2018, was still in good shape but Apple had since introduced [Diffable Data Sources](https://developer.apple.com/documentation/UIKit/updating-collection-views-using-diffable-data-sources), which would've been my go-to for a new project. In a corporate setting, such a refactor of "legacy code" would be considered technical debt and require prioritization and analysis. In my case, I just started a new branch and experimented. More on that in [Part 2]({% link projects/gunvault/_posts/2025-08-13-Implementing-Drag-and-Drop-in-UICollectionView-and-Core-Data–Part-2.md %}).