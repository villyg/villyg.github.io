---
layout: post
title: "Taming a Herd of View Controllers: Refactoring for Simplicity in Core Data with Swift"
date: 2025-07-31 17:37:16 -0400
tags: swift ios core-data
permalink: /posts/:title
---

- [Introduction](#introduction)
- [Data model](#data-model)
- [Workflow](#workflow)
- [Implementation](#implementation)
- [Problem](#problem)
- [Solution](#solution)
- [Conclusion](#conclusion)

## Introduction

If you’ve worked on an iOS app of any significant size, you’ve likely encountered it: the copy-paste monster. It starts innocently. You build a view controller, get it working perfectly, and then you need another one that’s almost the same. You duplicate the file, change a few lines, and move on. But then you need another, and another. Before you know it, you’re the reluctant owner of a herd of nearly identical view controllers, and any change to one means a tedious and error-prone update to all the others. To be fair - there is the [Rule of three (three strikes and you refactor)](https://en.wikipedia.org/wiki/Rule_of_three_(computer_programming)), so some temporary duplication is very much okay as long as it is addressed in a timely manner.



## Data model

Before diving into the details, it is worth talking about the data I am dealing with. This is an abbreviated entity diagram that shows some of it. Each `Gun` instance is a parent and it has multiple child attributes:  `Make`, `Caliber`, `Action`, `Finish`, `Type`, etc. Obviusly a `Gun` can only have one `Type` but a `Type` can be associated with multiple `Gun` objects. The attributes themselves have just one user-entered property: `Name`.

<pre class="mermaid">
   erDiagram
         Gun {
             NSDate manufactureDate
             String model
             String note
            String serialNumber
        }
        GunMake {
            String name
        }
        GunCaliber {
            String name
        }
        GunAction {
            String name
        }
        GunFinish {
            String name
        }
        GunType {
            String name
        }
        Gun }o--|| GunMake : "make"
        Gun }o--|| GunCaliber : "caliber"
        Gun }o--|| GunAction : "action"
        Gun }o--|| GunFinish : "finish"
        Gun }o--|| GunType : "type"

</pre>
<script src="https://cdn.jsdelivr.net/npm/mermaid@10.9.1/dist/mermaid.min.js"></script>

I am very well aware that there are better ways to organize parent-child attributes but given the current context and in the spirit of [KISS](https://en.wikipedia.org/wiki/KISS_principle) - this entity structure is sufficient and if need be - it can be migrated later to something better.

## Workflow

For the Gun-upserting workflow I started with a very basic approach. I settled on using a table view with all the Gun attributes shown in individual cells. Once a cell is selected eg. Type - another view would present itself with a Type list to pick from and a button to add a new one needed. The new Type entry screen is even simpler - just a single textbox to collect name of the Type attribute.

![Upsert view](/images/post-2025-08-01/Screenshot_01.png) 
![Selector view](/images/post-2025-08-01/Screenshot_02.png)
![New value view](/images/post-2025-08-01/Screenshot_03.png)


## Implementation

Although this is absolutely not a streamlined way of doing data entry - it has the benefit of being quick and easy to develop. A `GunUpsertController`, a `GunTypeEditController`, a `DBStringEditController` (not shown) along with some `Core Data` child branching logic to support cancellation was all that was needed. 

Note: this is partial implementation for brevity.


```swift
class GunUpsertController: UITableViewController {

    var gun: Gun!
    var managedObjectContext: NSManagedObjectContext!          
    
    override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        
        let result: UITableViewCell = UITableViewCell(style: .value1, reuseIdentifier: "gunType")
        result.accessoryType = .disclosureIndicator
        
        if indexPath.section == 0 {
            if indexPath.row == 0 { // Type
                result.textLabel?.text = Strings.LABEL_GUN_TYPE
                result.detailTextLabel?.text = gun.type?.name
            }          
        } 
        return result
    }
    
    
    override func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
    
        if indexPath.section == 0 {
            if indexPath.row == 0 { // Type
                let controller = DBGunTypeEditController(gun: gun)
                controller.editPropertyControllerDelegate = self    
                let navigationController = UINavigationController(rootViewController: controller)
                present(navigationController, animated: true, completion: nil)
            }
        } 
    }
    
}
```

```swift
class DBGunTypeEditController: UITableViewController {
    
    var gun: Gun!
    
    var managedObjectContext: NSManagedObjectContext!
    
    var editPropertyControllerDelegate: DBEditPropertyControllerDelegate?
    
    var validationHelper: ValidationHelper!
    
    lazy var fetchedResultsControler: NSFetchedResultsController<GunType> = {
        [unowned self] in
        let sortDescriptor = NSSortDescriptor(key: "name", ascending: true)
        
        let fReq: NSFetchRequest = GunType.fetchRequest()
        fReq.sortDescriptors = [sortDescriptor]
        
        let frc = NSFetchedResultsController(fetchRequest: fReq, managedObjectContext: self.managedObjectContext, sectionNameKeyPath: nil, cacheName: nil)
        
        return frc
        }()
    
    convenience init(gun: Gun) {
        self.init(style: UITableViewStyle.grouped)
        self.managedObjectContext = gun.managedObjectContext
        self.gun = gun
        self.validationHelper = ValidationHelper.sharedInstance          
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        self.tableView.register(UITableViewCell.self, forCellReuseIdentifier: "cell")
        
        let cancelButton = UIBarButtonItem(barButtonSystemItem: .cancel, target: self, action: #selector(cancelButtonTapped(sender:)))
        navigationItem.leftBarButtonItem = cancelButton
        
        let addBarButton = UIBarButtonItem(barButtonSystemItem: .add, target: self, action: #selector(addButtonTapped(sender:)))
        navigationItem.rightBarButtonItem = addBarButton
        
        navigationItem.title = Strings.TITLE_SELECT_GUN_TYPE
        
        fetchedResultsControler.delegate = self
        
        do {
            
            try fetchedResultsControler.performFetch()
            
        } catch {
            fatalError("Error fetching guns: \(error)")
        }
        
    }
    
    override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        
        return fetchedResultsControler.sections![section].numberOfObjects
        
    }    
    
    override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        
        guard let result = tableView.dequeueReusableCell(withIdentifier: "cell") else {
            return UITableViewCell()
        }
        
        let entity = fetchedResultsControler.object(at: indexPath)
        
        result.accessoryType = .disclosureIndicator
        result.textLabel?.text = entity.name
        
        return result
        
    }
    
    override func tableView(_ tableView: UITableView, titleForHeaderInSection section: Int) -> String? {
        
        if let fetchedObjects = fetchedResultsControler.fetchedObjects {
            if section == 0 && fetchedObjects.count > 0 {
                return Strings.MESSAGE_EDIT_GUN_TYPE_INSTRUCTIONS
            }
        }
        
        return nil
        
    }
    
    override func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        
        let selectedEntity = fetchedResultsControler.object(at: indexPath)
        
        gun.type = selectedEntity
        selectedEntity.addToGuns(gun)
        
        editPropertyControllerDelegate?.controllerDidFinish(self)
        
    }
    
    @objc func addButtonTapped(sender: UIBarButtonItem) {
        
        let childContext = NSManagedObjectContext(concurrencyType: .mainQueueConcurrencyType)
        childContext.parent = managedObjectContext
        let childEntity = GunType(context: childContext)
        
        let controller: DBStringEditController = DBStringEditController(managedObject: childEntity, forKey: "name")
        controller.delegate = self
        controller.navigationItem.title = Strings.TITLE_NEW_GUN_TYPE
        controller.textField.placeholder = Strings.LABEL_NAME
        let navigationController: UINavigationController = UINavigationController(rootViewController: controller)
        
        present(navigationController, animated: true, completion: nil)
        
    }
    
    @objc func cancelButtonTapped(sender: UIBarButtonItem) {
        
        editPropertyControllerDelegate?.contorllerDidCancel(self)
            
    }
}
```

## Problem

Obviously a gun has more than one attribute so I quickly replicated the code to add support for `Action` and `Caliber`. Unfortunately, I still had to account for `Make` and `Type` and I also wanted to add another module later for maintaining an `Ammo` inventory that would follow the same workflow so it was time to honor the [Rule of three](https://en.wikipedia.org/wiki/Rule_of_three_(computer_programming)) and refactor before things get out of control.


## Solution

The solution was to apply the [Don't Repeat Yourself (DRY)](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) principle by abstracting the common code into basic reusable classes. 

  The core logic in each replicated controller was responsible for:
   1. Setting up a `UITableView`.
   2. Configuring an `NSFetchedResultsController` to fetch and display a lookup dataset from Core Data.
   3. Handling table view updates via the `NSFetchedResultsControllerDelegate`.
   4. Implementing `DZNEmptyDataSetSource` and `DZNEmptyDataSetDelegate` to show a user-friendly message
      when the table was empty.
   5. Adding a "plus" button to the navigation bar to create a new entry.
   6. Implementing the swipe-to-delete functionality.

I decided to create a generic `DBEntityPropertyEditController`.

```swift
class DBEntityPropertyEditController<T: NSManagedObject>: UITableViewController, NSFetchedResultsControllerDelegate, DZNEmptyDataSetSource, DZNEmptyDataSetDelegate, DBEditPropertyControllerDelegate {
    
    var parentObject: NSManagedObject!
    var managedObjectContext: NSManagedObjectContext!
    var editPropertyControllerDelegate: DBEditPropertyControllerDelegate?
    var validationHelper: ValidationHelper!
    
    var propertyEntityName: String!
    var navigationItemTitle: String!
    var addNewTitle: String!
    var instructionsText: String!
    var parentRelationshipKeyPath: String!
    var propertyInverseRelationshipKeyPath: String!
    
    lazy var fetchedResultsControler: NSFetchedResultsController<T> = {
        [unowned self] in
        let sortDescriptor = NSSortDescriptor(key: "name", ascending: true)
        
        let fReq: NSFetchRequest<T> = NSFetchRequest<T>(entityName: self.propertyEntityName)
        fReq.sortDescriptors = [sortDescriptor]
        
        let frc = NSFetchedResultsController(fetchRequest: fReq, managedObjectContext: self.managedObjectContext, sectionNameKeyPath: nil, cacheName: nil)
        
        return frc
    }()
    
    
    convenience init(
        parentObject: NSManagedObject,
        propertyEntityName: String,
        navigationItemTitle: String,
        addNewTitle: String,
        instructionsText: String,
        parentRelationshipKeyPath: String,
        propertyInverseRelationshipKeyPath: String
    ) {
        self.init(style: UITableView.Style.grouped)
        self.managedObjectContext = parentObject.managedObjectContext
        self.parentObject = parentObject
        self.validationHelper = ValidationHelper.sharedInstance
        self.propertyEntityName = propertyEntityName
        self.navigationItemTitle = navigationItemTitle
        self.addNewTitle = addNewTitle
        self.instructionsText = instructionsText
        self.parentRelationshipKeyPath = parentRelationshipKeyPath
        self.propertyInverseRelationshipKeyPath = propertyInverseRelationshipKeyPath
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        self.tableView.register(UITableViewCell.self, forCellReuseIdentifier: "cell")
        self.tableView.emptyDataSetSource = self
        self.tableView.emptyDataSetDelegate = self
        
        let cancelButton = UIBarButtonItem(barButtonSystemItem: .cancel, target: self, action: #selector(cancelButtonTapped(sender:)))
        navigationItem.leftBarButtonItem = cancelButton
        
        let addBarButton = UIBarButtonItem(barButtonSystemItem: .add, target: self, action: #selector(addButtonTapped(sender:)))
        navigationItem.rightBarButtonItem = addBarButton
        
        navigationItem.title = navigationItemTitle
        
        fetchedResultsControler.delegate = self
        
        do {
            try fetchedResultsControler.performFetch()
        } catch {
            fatalError("Error fetching: \(error)")
        }
    }
    
    // MARK: - UITableViewDataSource
    
    override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        guard let sectionInfo = fetchedResultsControler.sections?[section] else  { fatalError("failed to resolve FRC") }
        return sectionInfo.numberOfObjects
    }
    
    override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        guard let result = tableView.dequeueReusableCell(withIdentifier: "cell") else {
            return UITableViewCell()
        }
        
        let entity = fetchedResultsControler.object(at: indexPath)
        
        result.accessoryType = .disclosureIndicator
        result.textLabel?.text = entity.value(forKey: "name") as? String
        
        return result
    }
    
    override func tableView(_ tableView: UITableView, titleForHeaderInSection section: Int) -> String? {
        if let fetchedObjects = fetchedResultsControler.fetchedObjects {
            if section == 0 && fetchedObjects.count > 0 {
                return instructionsText
            }
        }
        return nil
    }
    
    // MARK: - UITableViewDelegate
    
    override func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        let selectedEntity = fetchedResultsControler.object(at: indexPath)
        
        parentObject.setValue(selectedEntity, forKey: parentRelationshipKeyPath)
        
        let mutableGuns = selectedEntity.mutableSetValue(forKey: propertyInverseRelationshipKeyPath)
        mutableGuns.add(parentObject)

        AppDelegate().saveContext()
        
        editPropertyControllerDelegate?.controllerDidFinish(self)
    }
    
    @objc func addButtonTapped(sender: UIBarButtonItem) {
        let childContext = NSManagedObjectContext(concurrencyType: .mainQueueConcurrencyType)
        childContext.parent = managedObjectContext
        
        let childEntity = NSEntityDescription.insertNewObject(forEntityName: propertyEntityName, into: childContext)
        
        let controller = DBStringEditController(managedObject: childEntity, forKey: "name")
        controller.delegate = self
        controller.navigationItem.title = addNewTitle
        controller.textField.placeholder = Strings.LABEL_NAME
        let navigationController = UINavigationController(rootViewController: controller)
        
        present(navigationController, animated: true, completion: nil)
    }
    
    @objc func cancelButtonTapped(sender: UIBarButtonItem) {
        editPropertyControllerDelegate?.contorllerDidCancel(self)
    }

    // MARK: - DZNEmptyDataSetSource
    func title(forEmptyDataSet scrollView: UIScrollView!) -> NSAttributedString! {
        let title = Strings.MESSAGE_NO_ITEMS_PRESENT
        let attributes = [NSAttributedString.Key.font: UIFont.preferredFont(forTextStyle: .headline)]
        return NSAttributedString(string: title, attributes: attributes)
    }
    
    func description(forEmptyDataSet scrollView: UIScrollView!) -> NSAttributedString! {
        let title = Strings.MESSAGE_NO_ITEMS_PRESENT
        let attributes = [NSAttributedString.Key.font: UIFont.preferredFont(forTextStyle: .body)]
        return NSAttributedString(string: title, attributes: attributes)
    }

    // MARK: - DZNEmptyDataSetDelegate
    func emptyDataSetShouldDisplay(_ scrollView: UIScrollView!) -> Bool {
        return true
    }

    // MARK: - NSFetchedResultsControllerDelegate
    func controllerWillChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        tableView.beginUpdates()
    }

    func controller(_ controller: NSFetchedResultsController<NSFetchRequestResult>, didChange anObject: Any, at indexPath: IndexPath?, for type: NSFetchedResultsChangeType, newIndexPath: IndexPath?) {
        switch type {
        case .insert:
            tableView.insertRows(at: [newIndexPath!], with: .fade)
        case .delete:
            tableView.deleteRows(at: [indexPath!], with: .fade)
        case .update:
            tableView.reloadRows(at: [indexPath!], with: .fade)
        case .move:
            tableView.moveRow(at: indexPath!, to: newIndexPath!)
        @unknown default:
            fatalError()
        }
    }

    func controllerDidChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        tableView.endUpdates()
    }
    
    // MARK: - DBEditPropertyControllerDelegate
    func controllerDidFinish(_ controller: UIViewController) {
        if let context: NSManagedObjectContext = (controller as! DBStringEditController).managedObjectContext, context.hasChanges {
            do {
                try context.save()
                dismiss(animated: true, completion: nil)
            } catch {
                let nserror = error as NSError
                validationHelper.showCoreDataValidationErrors(error: nserror, controller: controller)
            }
        }
    }
    
    func contorllerDidCancel(_ controller: UIViewController) {
        controller.dismiss(animated: true)
    }
}
```

## Benefits

- I went from multiple copies of almost the same code to only one class.
- I deleted numerous unit tests that were no longer needed.
- I left the door open for further subclassing and customization while still maintaining a very tight codebase. In example - I could derive a separate controller that deals only with `Gun` attributes or one only with `Ammo` attributes.

```swift
class GunPropertyEditController<T: NSManagedObject>: DBEntityPropertyEditController<T> {
    
    convenience init(
        gun: Gun,
        entityName: String,
        navigationItemTitle: String,
        addNewTitle: String,
        instructionsText: String,
        gunRelationshipKeyPath: String
    ) {
        self.init(
            parentObject: gun,
            propertyEntityName: entityName,
            navigationItemTitle: navigationItemTitle,
            addNewTitle: addNewTitle,
            instructionsText: instructionsText,
            parentRelationshipKeyPath: gunRelationshipKeyPath,
            propertyInverseRelationshipKeyPath: "guns"
        )
    }
}
```

I could also go a step further and subclass one more time per attribute in case I wanted to do something different.

```swift
class GunTypeEditController: GunPropertyEditController<GunType> {
    convenience init(gun: Gun) {
        self.init(
            gun: gun,
            entityName: "GunType",
            navigationItemTitle: Strings.TITLE_SELECT_GUN_TYPE,
            addNewTitle: Strings.TITLE_NEW_GUN_TYPE,
            instructionsText: Strings.MESSAGE_EDIT_GUN_TYPE_INSTRUCTIONS,
            gunRelationshipKeyPath: "type"
        )
    }
}
```
Whatever the reason might be, this new base `DBEntityPropertyEditController` would handle all the boilerplate work, leaving the subclasses with the minimal task of providing their specific configurations.

## Conclusion

Taking the time to refactor duplicated code is always a worthwhile investment. It pays dividends in reduced complexity, improved stability, and faster development down the road. So next time you find yourself reaching for copy-paste, take a moment to consider if a base class or another form of abstraction could save you from a future maintenance nightmare. Your future self will thank you.
