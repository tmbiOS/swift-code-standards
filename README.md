# Swift Team Code Standards

- [Swift Team Code Standards](#swift-team-code-standards)
  * [Integrated development environment](#integrated-development-environment)
  * [iOS architecture](#ios-architecture)
    + [Posts](#posts)
  * [App architecture](#app-architecture)
    + [Development flow](#development-flow)
    + [Code structure](#code-structure)
      - [The rule of threes](#the-rule-of-threes)
    + [Models](#models)
      - [Making models expressible by string interpolation.](#making-models-expressible-by-string-interpolation)
    + [Posts](#posts-1)
  * [Tips](#tips)
    + [Architecture](#architecture)
      - [Using child view controllers as plugins in Swift](#using-child-view-controllers-as-plugins-in-swift)
      - [Reusable data sources in Swift](#reusable-data-sources-in-swift)
      - [TableViewCellPresenter](#tableviewcellpresenter)
  * [Tools](#tools)
    + [Generate Xcode project](#generate-xcode-project)
    + [Generate documentation](#generate-documentation)
    + [Coverage reports](#coverage-reports)

## Integrated development environment

* Use latest **Xcode** version (**Xcode 11**).

## iOS architecture

* Use **VIP (clean-swift cycle)** for complex scenes.
* Use **MVP** architecture for small scenes (less than 300 line of code).

### Posts

[Clean Swift iOS Architecture for Fixing Massive View Controller](http://clean-swift.com/clean-swift-ios-architecture/ "Clean Swift iOS Architecture for Fixing Massive View Controller")

## App architecture

* Use Modular architecture and Swift Package Manager.

### Development flow
-   Having the setup already in place, you may be wondering what development flow you could use while developing in SPM. This is what works for me right now:
-   Pull the changes from remote and generate project using `swift build & swift package generate-xcodeproj`.
-   Implement features/fixes and push it to the remote. When in need of another dependency, add it to the **Package.swift** and regenerate the project.
- Repeat steps 1 and 2.

### Code structure

#### The rule of threes
Extract 3 or more properties or local variables (that share the same prefix) into their own method, type or tuple.

### Models

#### Making models expressible by string interpolation.

```swift
struct Path {
    var string: String
}

func loadFile(at path: Path) throws -> File {
    ...
}

extension Path: ExpressibleByStringLiteral {
    init(stringLiteral value: String) {
        self.string = value
    }
}

extension Path: ExpressibleByStringInterpolation {}

try loadFile(at: "~/documents/article.md")
```

### Posts

[Making types expressible by string interpolation](https://www.swiftbysundell.com/tips/making-types-expressible-by-string-interpolation/ "Making types expressible by string interpolation")

[Logic controllers in Swift](https://www.swiftbysundell.com/articles/logic-controllers-in-swift/ "Logic controllers in Swift")

[Using child view controllers as plugins in Swift](https://www.swiftbysundell.com/articles/using-child-view-controllers-as-plugins-in-swift/)

[Reusable data sources in Swift](https://www.swiftbysundell.com/articles/reusable-data-sources-in-swift/)

[Preventing views from being model aware in Swift](https://www.swiftbysundell.com/articles/preventing-views-from-being-model-aware-in-swift/)

## Tips

### Architecture

#### Using child view controllers as plugins in Swift

Let's make an extension on `UIViewController` that makes handling child view controllers a lot simpler:

```swift
extension UIViewController {
    func add(_ child: UIViewController) {
        addChild(child)
        view.addSubview(child.view)
        child.didMove(toParent: self)
    }

    func remove() {
        // Just to be safe, we check that this view controller
        // is actually added to a parent before removing it.
        guard parent != nil else {
            return
        }

        willMove(toParent: nil)
        view.removeFromSuperview()
        removeFromParent()
    }
}
```

Child view controller (`LoadingViewController`) that showing an activity indicator while loading data:

```swift
class LoadingViewController: UIViewController {
    private lazy var activityIndicator = UIActivityIndicatorView(activityIndicatorStyle: .gray)

    override func viewDidLoad() {
        super.viewDidLoad()

        activityIndicator.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(activityIndicator)

        NSLayoutConstraint.activate([
            activityIndicator.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            activityIndicator.centerYAnchor.constraint(equalTo: view.centerYAnchor)
        ])
    }

    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)

        // We use a 0.5 second delay to not show an activity indicator
        // in case our data loads very quickly.
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.5) { [weak self] in
            self?.activityIndicator.startAnimating()
        }
    }
}
```

`ErrorViewController` that displays an error message and has a `Reload button` with `reloadHandler`:

```swift
class ErrorViewController: UIViewController {
    var reloadHandler: () -> Void = {}
}

private extension ListViewController {
    func handle(_ error: Error) {
        let errorViewController = ErrorViewController()

        errorViewController.reloadHandler = { [weak self] in
            self?.loadItems()
        }

        add(errorViewController)
    }
}
```

Using a child view controller: 

```swift
class ListViewController: UITableViewController {
    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        loadItems()
    }

    private func loadItems() {
        let loadingViewController = LoadingViewController()
        add(loadingViewController)

        dataLoader.loadItems { [weak self] result in
            loadingViewController.remove()
            self?.handle(result)
        }
    }
}
```

#### Reusable data sources in Swift

```swift
class TableViewDataSource<Model>: NSObject, UITableViewDataSource {
    typealias CellConfigurator = (Model, UITableViewCell) -> Void

    var models: [Model]

    private let reuseIdentifier: String
    private let cellConfigurator: CellConfigurator

    init(models: [Model],
         reuseIdentifier: String,
         cellConfigurator: @escaping CellConfigurator) {
        self.models = models
        self.reuseIdentifier = reuseIdentifier
        self.cellConfigurator = cellConfigurator
    }
  
    func tableView(_ tableView: UITableView,
           numberOfRowsInSection section: Int) -> Int {
    return models.count
  }

  func tableView(_ tableView: UITableView,
           cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let model = models[indexPath.row]
    let cell = tableView.dequeueReusableCell(
      withIdentifier: reuseIdentifier,
      for: indexPath
    )

    cellConfigurator(model, cell)

    return cell
  }
}

```

Using `TableViewDataSource`  with 1 section:

```swift
extension TableViewDataSource where Model == Message {
    static func make(for messages: [Message],
                     reuseIdentifier: String = "message") -> TableViewDataSource {
        return TableViewDataSource(
            models: messages,
            reuseIdentifier: reuseIdentifier
        ) { (message, cell) in
            cell.textLabel?.text = message.title
            cell.detailTextLabel?.text = message.preview
        }
    }
}

func messagesDidLoad(_ messages: [Message]) {
    dataSource = .make(for: messages)
    tableView.dataSource = dataSource
}
```

Composing sections

```swift
class SectionedTableViewDataSource: NSObject {
    private let dataSources: [UITableViewDataSource]

    init(dataSources: [UITableViewDataSource]) {
        self.dataSources = dataSources
    }
}

extension SectionedTableViewDataSource: UITableViewDataSource {
    func numberOfSections(in tableView: UITableView) -> Int {
        return dataSources.count
    }

    func tableView(_ tableView: UITableView,
                   numberOfRowsInSection section: Int) -> Int {
        let dataSource = dataSources[section]
        return dataSource.tableView(tableView, numberOfRowsInSection: 0)
    }

    func tableView(_ tableView: UITableView,
                   cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let dataSource = dataSources[indexPath.section]
        let indexPath = IndexPath(row: indexPath.row, section: 0)
        return dataSource.tableView(tableView, cellForRowAt: indexPath)
    }
}
```

Using SectionedTableViewDataSource with 2 sections:
```swift
let dataSource = SectionedTableViewDataSource(dataSources: [
    TableViewDataSource.make(for: recentContacts),
    TableViewDataSource.make(for: topMessages)
])
```

#### TableViewCellPresenter

Preventing views from being model aware in Swift:

```swift
class UserTableViewCellPresenter {
    private let friendManager: FriendManager

    init(friendManager: FriendManager) {
        self.friendManager = friendManager
    }

    func configure(_ cell: UITableViewCell, forDisplaying user: User) {
        cell.textLabel?.text = "\(user.firstName) \(user.lastName)"
        cell.imageView?.image = user.profileImage

        if !user.isFriend {
            // We create a local reference to the friend manager so that
            // the button doesn't have to capture the configurator.
            let friendManager = self.friendManager

            let addFriendButton = AddFriendButton()

            addFriendButton.closure = {
                friendManager.addUserAsFriend(user)
            }

            cell.accessoryView = addFriendButton
        } else {
            cell.accessoryView = nil
        }
    }
}

class UserListViewController: UITableViewController {
    override func tableView(_ tableView: UITableView,
                            cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
        let user = users[indexPath.row]

        presenter.configure(cell, forDisplaying: user)

        return cell
    }
}
```

**View factories**

For "static" views it's often enough to be able to configure them once. 
In such situations, using the *Factory* pattern can be a great option
```swift
class MessageViewFactory {
    func makeView(for message: Message) -> UIView {
        let view = TextView()

        view.titleLabel.text = message.title
        view.textLabel.text = message.text
        view.imageView.image = message.icon

        return view
    }
}
```


## Tools

### Generate Xcode project

A Swift command line tool for generating your Xcode project

[XcodeGen](https://github.com/yonaskolb/XcodeGen "XcodeGen")

### Generate documentation

Soulful docs for Swift

[jazzy](https://github.com/realm/jazzy)

### Coverage reports

External tool: [Codecov](https://codecov.io/)
