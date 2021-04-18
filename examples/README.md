# Examples

_Section for examples._

## Design patterns

_Section for design patterns._

### Using the Switch(true) Pattern

_Switch true w/ abstracted validation criteria._

```js
const user = {
  firstName: 'Seán',
  lastName: 'Barry',
  email: 'my.address@email.com',
  number: '00447123456789',
};

function assertValidUserData(userDataObj) {
  switch (true) {
    case !isDefined(userDataObj):
      throw new AssertionError('User must be defined.');
    case !isString(userDataObj.firstName):
      throw new AssertionError("User's first name must be a string");
    case !isValidEmail(userDataObj.email):
      throw new AssertionError(
        "User's email address must be a valid email address"
      );
    case !isValidPhoneNumber(userDataObj.number):
      throw new AssertionError(
        "User's phone number must be a valid phone number"
      );
    // any number of other validation cases here
    default:
      return userDataObj;
  }
}

const validUserData = assertValidUserData(user);
```

[Attribution](https://seanbarry.dev/posts/switch-true-pattern)
