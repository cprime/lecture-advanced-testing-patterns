# Patterns for Better Testing

* These aren't only for MVVM but they will come in handy

# Unit Testing Commandments

For tests to be useful, they must:

* Run Often
  * Tests need to be run often so that they catch bugs as they are introduced
* Run Quickly
  * Tests need to run quickly so that they can actually be run often without slowing development
* Run Reliably
  * Tests should only fail if there is a bug in the system under test
* Be Readable
  * Failing tests that are difficult to read will be even harder to fix

# Testing With Dependencies

```
func searchFor(searchTerm: String, completion: @escapable (Result<[Album]>) -> Void) {
  RestClient.shared.searchFor(searchTerm: searchTerm) { (result: Result<[Album]>) in
    switch result {
      case let .success(albums):
        self.albums = albums
      case let .failure(error):
        self.albums = []
    }
  }
}
```

* Test API request is made (without networking)
* On success
  * .albums is updated
* On failure
  * .albums is cleared

## Mocking

* Adheres to the same interface as dependency
* Configurable for our test cases

### Mocking in Swift: Protocols

```
protocol RestClient {
  func searchFor(searchTerm: searchTerm, completion: @escapable (Result<[Album]>) -> Void)
}

class LillyRestClient: RestClient {
  func searchFor(searchTerm: searchTerm, completion: @escapable (Result<[Album]>) -> Void) {
    ...
  }
}

class MockRestClient: RestClient {
  func searchFor(searchTerm: searchTerm, completion: @escapable (Result<[Album]>) -> Void) {
    ...
  }
}
```

## Mock Methods

### Methods without return values

```
class MockLogger: Logger {
  func log(_ message: String) {
  }
}
```

### Methods with return values

```
class MockDataStore: DataStore {
  var stubbedUserList = [User]()
  func getUserList() -> [User] {
    return stubbedUserList
  }
}
```

### Methods with multiple return values

```
class MockValueTransformer: ValueTransformer {
  var stubbedTransformedValues = [String:Date]()
  func stringToDate(_ value: String) -> Date {
    return stubbedTransformedValues[value]!
  }
}
```

### Asynchronous methods

```
class MockRestClient: RestClient {
  var stubbedSearchResult = Result.failure(RestError())
  func searchFor(searchTerm: searchTerm, completion: @escapable (Result<[Album]>) -> Void) {
    DispatchQueue.main.async {
      completion(stubbedSearchResult)
    }
  }
}
```

_If the original method is asynchronous, the mocked should be asynchronous_

## Spying

```
// MockDataStore

var clearTransactionCount = 0
func clearTransations() {
  clearTransactionCount += 1
}

var capturedSavedUsers = [User]()
func save(_ user: User) {
  capturedSavedUsers.append(user)
}

var capturedUpdateArguments = [(User, FormData)]()
func update(_ user: User, with formData: FormData) {
  capturedUpdateArguments.append(user)
}
```

* Ensure that our units are interacting with their dependencies as expected
* 0-arg, 1-arg, n-arg variants

# Dependency Injection

* Use the mocked version in our tests instead of the real version

## Parameter Injection

```
class SearchViewModel {
  let restClient: RestClient

  init(restClient: RestClient) {
    self.restClient = restClient
  }

  func searchFor(searchTerm: String, completion: @escapable (Result<[Album]>) -> Void) {
    restClient.searchFor(searchTerm: searchTerm) { ... }
  }
}
```

## Default Injection

```
class SearchViewModel {
  let restClient: RestClient

  init(restClient: RestClient = LillyRestClient.shared) {
    self.restClient = restClient
  }
}
```

## AppConfigurator

```
class AppConfigurator {
  static var restClient = LillyRestClient()
}
```

```
SearchViewModel(restClient: AppConfigurator.restClient)
```

## DI and Mocking Together

### Spying

```
func testSearchResultsWithTrimmedTerm() {
  mockRestClient.stubbedSearchResult = .success(albums)

  viewModel.searchFor(searchTerm: " bob ") { _ in }

  XCTAssertEqual(mockRestClient.capturedSearchArguments, ["bob"])
}
```

### Mocking Happy Path

```
func testSearchUpdatesListOnSuccess() {
  let viewModel = SearchViewModel(restClient: mockRestClient)
  mockRestClient.stubbedSearchResult = .success(albums)

  let expectation = expectation(description: "request completes")
  viewModel.searchFor(searchTerm: "bob") { result in
    XCTAssertTrue(result.isSuccess)
    XCTAssertEqual(viewModel.albums, albums)
    expectation.fulfill()
  }

  waitForExpectations([expectation])
}
```

### Mocking Sad Path

```
func testSearchUpdatesErrorListOnFailure() {
  let viewModel = SearchViewModel(restClient: mockRestClient)
  mockRestClient.stubbedSearchResult = .failure(RestError(404))

  let expectation = expectation(description: "request completes")
  viewModel.searchFor(searchTerm: "bob") { result in
    XCTAssertTrue(result.isFailure)
    XCTAssertEqual(viewModel.albums, [])
    expectation.fulfill()
  }

  waitForExpectations([expectation])
}
```

## Additional Thoughts on Mocking and Spying

* Mocking through protocols is additional boiler plate
  * Forces you to limit your dependencies to small well-defined interfaces
  * Lighter weight than including a mocking framework like OCMock that is ObjC
* Mocks vs Stubs vs Fakes vs Spies vs Dummies
  * Slightly different semantics
  * All stand in for dependencies
* Be careful not to put too much logic into your mocks
  * Make sure you are testing your units, not your testing tools
  * Implement the bare minimum to get your dependencies out of the way

# Testing UIKit Delegate Implementations

* Migrate their logic into VMs (without view references)

**insert UITextFieldDelegate example**

# Testing Custom Delegate Implementations

```
class MessageComposerDelegate {
  func updateMessageText(_ text: String) {
    messageText = text
  }

  func shouldChangeText(in range: NSRange, replacementText text: String) -> Bool {
      let afterText = (messageText as NSString).replacingCharacters(in: range, with: text)
      return afterText.count <= 140
  }
}
```

```
extension MessageComposerViewController: UITextViewDelegate {
    func textView(_ textView: UITextView, shouldChangeTextIn range: NSRange, replacementText text: String) -> Bool {
        return viewModel.shouldChangeText(in: range, replacementText: text)
    }

    func textViewDidChange(_ textView: UITextView) {
        viewModel.updateMessageText(textView.text ?? "")
    }
}
```

```
func testShouldUpdateTextWhenUpdatedTextIsLessThanLimit() {
    viewModel.updateMessageText("")
    XCTAssertEqual(viewModel.shouldChangeText(in: 0..<0, replacementText: "1"), true)

    viewModel.updateMessageText("123")
    XCTAssertEqual(viewModel.shouldChangeText(in: 0..<0, replacementText: "1"), true)

    viewModel.updateMessageText(fullLengthMessage)
    XCTAssertEqual(viewModel.shouldChangeText(in: 0..<1, replacementText: "1"), true)
}
```

```
func testShouldNotUpdateTextWhenUpdatedTextIsLargerThanLimit() {
    viewModel.updateMessageText("1")
    XCTAssertEqual(viewModel.shouldChangeText(in: 0..<0, replacementText: fullLengthMessage), false)

    viewModel.updateMessageText(fullLengthMessage)
    XCTAssertEqual(viewModel.shouldChangeText(in: 0..<0, length: 0), replacementText: "1"), false)
}
```

* Use this same approach with your own custom delegate protocols

# Higher Level Assertions

```

func testEventCreatedAfterDeadlineIsMissed() {
  timeProvider.currentDate = dueDate
  viewModel = EventViewModel(dueDate: dueDate)

  assertShowMissedState(viewModel)
}

func testIdleEventIsMissedAfterDeadline() {
  timeProvider.currentDate = Date.distantPast
  viewModel = EventViewModel(dueDate: dueDate)

  timeProvider.currentDate = dueDate
  viewModel.onMinuteTick()

  assertShowMissedState(viewModel)
}

private func assertShowMissedState(_ viewModel: EventViewModel, file: StaticString = #file, line: UInt = #line) {
  XCTAssertTrue(viewModel.actionButtonHidden, file: file, line: line)
  XCTAssertEqual(viewModel.statusText, "Missed", file: file, line: line)
  XCTAssertEqual(viewModel.statusColor, UIColor.red, file: file, line: line)
}

```

# Disable Application During Testing

# Networking Testing

* Abstract networking requests behind a networking abstraction layer

**insert example**

## OHHTTPStubs

# View Events (works best with Rx)

# Why do we write unit Tests

* Testing catches regression bugs before they happen, allowing you can integrate new code faster and with more confidence
* Manually testing edge cases often takes a long time to setup and perform, and those actions must be done every time you want to test them, automated edge case tests only need to be setup once and can be run over and over
* Testing forces you to write maintainable code because it is generally harder to test poorly designed code
* Good tests are effective documentation making it easier to introduce new developers to a project

## Additional Resources

* https://github.com/IntrepidPursuits/sherpa/blob/master/ios/ios_testing_references.md
* https://medium.com/@hesham.salman/the-ios-testing-manifesto-e1bc821cc4c3
* https://nsscreencast.com/series/13-testing-ios-applications
