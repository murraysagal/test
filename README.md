## Description

This project demonstrates that Core Data validation can be implemented on iOS with very little effort and that it works well with one caveat. The problem is that, by default, the error messages returned are not directly consumable. If you are willing to deviate from a KVC compliant approach this problem is easy to work around, and is shown below. If you must adhere to KVC this can be done too with a little extra code.

All the details are below but here's the summary. If you must remain KVC compliant the only way to get a consumable error message to the view controller is to remove the validation for an NSManagedObject's properties from the Core Data model editor and implement validation using Core Data's `validate<key>:error:` method.

The last section puts forward two possible enhancements to Core Data that could eliminate the problem. I'm planning to create a radar for this issue but I'd love to get some feedback first.

## Scenario

A new Person object, an `NSManagedObject` subclass, is inserted into the MOC. The view controller displays a form for editing. Early (before save) validation is implemented using the standard KVC `validateValue:forKey:error:` method like this...

```Objective-C
NSError *error;
BOOL isValid = [person validateValue:&firstName forKey:@"firstName" error:&error];
if (!isValid) { /* handle the error here */ }
```

Validation constraints, like min and max width, are set in Core Data's model editor in Xcode. 

## The Problem

When `firstName` is validated and it's too short an error like this is returned...

```Objective-C
Error Domain=NSCocoaErrorDomain Code=1670 "The operation couldn’t be completed. (Cocoa error 1670.)" UserInfo=0x8f44a90 {NSValidationErrorObject=<Event: 0xcb41a60> (entity: Event; id: 0xcb40d70 <x-coredata://ADB90708-BAD9-47D8-B722-E3B368598E94/Event/p1> ; data: {
    firstName = B;
    }), NSValidationErrorKey=firstName, NSLocalizedDescription=The operation couldn’t be completed. (Cocoa error 1670.), NSValidationErrorValue=B}
```

You can see that localizedDescription is not suitable for displaying the error to the user. But the error code is there so it is straightforward to implement something like this...

```Objective-C
switch ([error code]) {

    case NSValidationStringTooShortError:
        errorMsg = @"First name must be at least two characters.";
        break;
               
    case NSValidationStringTooLongError:
        errorMsg = @"First name is too long.";
        break;

    // of course, for real, these would be localized strings, not just hardcoded like this
}
```

This is good in concept but `firstName`, and other `Person` properties, is editable on other view controllers so that switch would have to be implemented again on whatever view controller edits `firstName`. Or, of course, it could also be implemented as a method in the `Person` class, something like `localizedErrorMessageForKey:withCode:`...

```Objective-C
NSError *error;
BOOL isValid = [person validateValue:&firstName forKey:@"firstName" error:&error];
if (!isValid) { 
	NSString *errorMessage = [person localizedErrorMessageForKey:@"firstName" code:[error code]];
	...
 }
```

But, however it's implemented, getting a consumable error message to the view controller requires the developer to deviate in some way from the standard KVC approach. This may be acceptable in some cases but assume for the moment that there is a use case that requires strict adherence to the standard KVC approach. 

So, is there a way to return a directly consumable error message *and* adhere to KVC? In other words, to not require the switch statement or something like `localizedErrorMessageForKey:withCode:`?

## Adhering to KVC

Looking at the Core Data docs for Property-Level Validation reveals this...

> If you want to implement logic in addition to the constraints you provide in the managed object model, you should not override validateValue:forKey:error:. Instead you should implement methods of the form `validate<Key>:error:`. 

So now `validateFirstName:error:` is partially implemented in `Person.m` like this allowing ioValue and `outError` to be inspected...

```Objective-C
- (BOOL)validateFirstName:(id *)ioValue error:(NSError **)outError {
    NSLog(@"*ioValue= %@", *ioValue);
    NSLog(@"*outError= %@", *outError);
}
```

But inside `validateFirstName:error:`, `outError` is still nil even when `firstName` is invalid. When control returns to the view controller there is an error like at the top of this question indicating that the Core Data validation runs *after* any `validate<key>:error:` implementations but, again, that's too late.

## Workaround

In the current implementation of Core Data I think there may be only one way to return a consumable error message *and* remain within KVC. 

- Remove all the validation from the Core Data model editor in Xcode and perform all of the validation in the `validate<key>:error:` methods like `validateFirstName:error:`. If validation fails create a new `NSError` object with a consumable error message and return that to the view controller. Here's an example:

```Objective-C
- (BOOL)validateFirstName:(id *)ioValue error:(NSError **)outError {
    
    // firstName's validation is not specified in the model editor, it's specified here.
    // field width: min 2, max 10
    
    BOOL isValid = YES;
    NSString *firstName = *ioValue;
    NSString *errorMessage;
    NSInteger code;
    
    if (firstName.length < 2) {
        
        errorMessage = @"First Name must be at least 2 characters.";
        code = NSValidationStringTooShortError;
        isValid = NO;
        
    } else if (firstName.length > 10) {
        
        errorMessage = @"First Name can't be more than 10 characters.";
        code = NSValidationStringTooLongError;
        isValid = NO;
        
    }
    
    if (outError && errorMessage) {
        NSDictionary *userInfo = @{ NSLocalizedDescriptionKey : errorMessage };
        NSError *error = [[NSError alloc] initWithDomain:@"test"
                                                    code:code
                                                userInfo:userInfo];
        *outError = error;
    }
    
    return isValid;
    
}
```

## Strict KVC Use Case

In my case I can't deviate from KVC because I'm creating an editing framework. Within the framework, all the view controller knows about the property being edited is the key, like `@"firstName"` and its model object, like `self.person`. Thus it can't do anything other than the standard KVC approach to validation.

The workaround works fine when you're starting a new project. But users of the framework who are already using Core Data probably have constraints specified in the model and adding my framework would mean moving all of that to the validation method.

# Suggestions for Enhancing Core Data

I have two suggestions for enhancing Core Data on iOS that would, I think, alleviate this problem altogether. I prefer the first suggestion. 

1. In the Xcode Core Data model editor allow the developer to specify the error message along with the constraint. Of course, these wouldn't be hardcoded strings but keys to a localized message. It should also support substitution of the constraint value so the strings wouldn't need to be changed if the value changed. "First name must be at least 2 characters." The 2 would be substituted. This way, in most cases, developers would not have to implement `validate<key>:error:` methods because Core Data would use the developer-provided error messages in the error object. 

2. Have Core Data perform the validation for the constraints specified in the model before calling `validate<key>:error:`. If there is an error or errors pass along a filled in error object. Then the developer can inspect `outError` and create a new error object that contains a directly consumable error message and return that to the view controller.
