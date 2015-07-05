[![Build Status](https://travis-ci.org/nicklockwood/AutoCoding.svg)](https://travis-ci.org/nicklockwood/AutoCoding)


Purpose
--------------

AutoCoding is a category on NSObject that provides automatic support for NSCoding to any object. This means that rather than having to implement the `initWithCoder:` and `encodeWithCoder:` methods yourself, all the model classes in your app can be saved or loaded from a file without you needing to write any additional code.

Of course no automated system can read your mind, so AutoCoding does place certain restrictions on how you design your classes; For example, you should avoid using structs that are not already NSCoding-compliant via NSValue.

Use of AutoCoding is by no means and all-or-nothing decision. You are free to implement your own NSCoding or NSCopying methods on any class in your project and they will simply override the automatically generated methods.


Supported OS & SDK Versions
-----------------------------

* Supported build target - iOS 8.1 / Mac OS 10.10 (Xcode 6.1, Apple LLVM compiler 6.0)
* Earliest supported deployment target - iOS 5.0 / Mac OS 10.7
* Earliest compatible deployment target - iOS 4.3 / Mac OS 10.6

NOTE: 'Supported' means that the library has been tested with this version. 'Compatible' means that the library should work on this OS version (i.e. it doesn't rely on any unavailable SDK features) but is no longer being tested for compatibility and may require tweaking or bug fixes to run correctly.


ARC Compatibility
------------------

AutoCoding is compatible with both ARC and non-ARC compile targets.


Thread Safety
--------------

AutoCoding is fully thread-safe.


Installation
--------------

To use the AutoCoding category in your project, just drag the AutoCoding.h and .m files into your project.


Security
-------------------

As of version 2.0, AutoCoding supports the NSSecureCoding protocol automatically, and returns YES for the `+supportsSecureCoding` method for all objects by default. In addition to this, assuming you do not override `+supportsSecureCoding` to return NO, AutoCoding will automatically throw an exception when attempting to decode a class whose type doesn't match the property it is being assigned to. This makes it much harder for a malicious party to craft an NSCoded  file that, when loaded by your app, will cause it to execute code that you didn't intend it to.


Category methods
-----------------------------

The NSObject(AutoCoding) category extends NSObject with the following methods. Since this is a category, every single Cocoa object, including AppKit/UIKit objects and BaseModel instances inherit these methods.

    + (NSDictionary *)codableProperties;

This method returns a dictionary containing the names and classes of all the properties of the class that will be automatically saved, loaded and copied when the object is archived using NSKeyedArchiver/Unarchiver. The values of the dictionary represent the class used to encode each property (e.g. NSString for strings, NSNumber for numeric values or booleans, NSValue for structs, etc).

This dictionary is automatically generated by scanning the properties defined in the class definition at runtime. Read-only and private properties will also be coded as long as they have KVC-compliant ivar names (i.e. the ivar matches the property name, or is the same but with a _ prefix). Any properties that are not backed by an ivar, or whose ivar name does not match the property name will not be encoded (this is a design feature, not a limitation - it makes it easier to exclude properties from encoding)

It is not normally necessary to override this method unless you wish to add ivars for coding that do not have matching property definitions, or if you wish to code virtual properties (properties or setter/getter method pairs that are not backed by an ivar). If you wish to exclude certain properties from the serialisation process, you can return them in the `uncodableProperties` method and they will be automatically removed from this dictionary.

Note that this method only returns the properties defined on a particular class and not any properties that are inherited from its superclasses. You *do not* need to call `[super codableProperties]` if you override this method.

    - (NSDictionary *)codableProperties;
    
This method returns all the codable properties of the object, including those that are inherited from superclasses. You should not override this method - if you want to add additional properties, override the `+codableProperties` class method instead.

    - (NSDictionary *)dictionaryRepresentation;

This method returns a dictionary of the values of all the codable properties of the object. It is equivalent to calling `dictionaryWithValuesForKeys:` with the result of `[[object codableProperties] allKeys]` as the parameter.

    - (void)setWithCoder:(NSCoder *)coder;
    
This method populates the object's properties using the provided coder object, based on the codableKeys array. This is called internally by the initWithCoder: method, but may be useful if you wish to initialise an object from a coder archive after it has already been created. You could even initialise the object by merging the results of several different archives by calling `setWithCoder:` more than once.

    + (instancetype)objectWithContentsOfFile:(NSString *)path;
    
This attempts to load the file using the following sequence: 1) If the file is an NSCoded archive, load the root object and return it, 2) If the file is an ordinary Plist, load and return the root object, 3) Return the raw data as an NSData object. If the de-serialised object is not a subclass of the class being used to load it, an exception will be thrown (to avoid this, call the method on `NSObject` instead of a specific subclass).
    
    - (BOOL)writeToFile:(NSString *)filePath atomically:(BOOL)useAuxiliaryFile;
    
This attempts to write the file to disk. This method is overridden by the equivalent methods for NSData, NSDictionary and NSArray, which save the file as a human-readable XML Plist rather than an binary NSCoded Plist archive, but the `objectWithContentsOfFile:` method will correctly de-serialise these again anyway. For any other object it will serialise the object using the NSCoding protocol and write out the file as a NSCoded binary Plist archive. Returns YES on success and NO on failure.


NSCopying
------------------

As of version 2.1, NSCopying is no longer implemented automatically, as this caused some compatibility problems with Core Data NSManagedObjects. If you wish to implement copying, this can be done quite easily by looping over the codableProperties keys and copying those properties individually to a new instance of the object (as follows):

    - (id)copyWithZone:(id)zone
    {
        id copy = [[[self class] alloc] init];
        for (NSString *key in [self codableProperties])
        {
            [copy setValue:[self valueForKey:key] forKey:key];
        }
        return copy;
    }
    
In order to properly support NSCopying, you should also override the `-hash` and `-isEqual:` methods for any object you intend to use with copying, so that a copied object has the same hash value and is equal to the original.


Tips
--------------------------------------

1. To exclude certain properties of your object from being encoded, you can do so in any of the following ways:

    * Only use an ivar, without declaring a matching @property.
    * Change the name of the ivar to something that is not KVC compliant (i.e. not the same as the property, or the property name with an _ prefix). You can do this using the @synthesize method, e.g. @synthesize foo = unencodableFoo;
    * Override the +codableProperties method

2. If you want to perform initialisation of the class post or prior to the properties being loaded via NSCoding, override the `setWithCoder:` method and call the super-implementation before or after applying your own logic, like this:

        - (void)setWithCoder:(NSCoder *)coder
        {
            //pre-initialisation
            [super setWithCoder:coder];
            //post-initialisation
        }

    Note that unlike in previous versions, the `init` method is not called when using `initWithCoder:`.

3. If you want to perform some cleanup or post-processing or substitute a different object after the object has been loaded via NSCoding, you can use the `awakeAfterUsingCoder:` method, which is defined in the NSObject class reference.

4. You can add additional coding/decoding logic by overriding the `setWithCoder:` and/or `encodeWithCoder:` methods. As long as you call the [super ...] implementation, the auto-coding will still function.

5. If you wish to substitute a different class for properties of a given type - for example if you have changed the name of a class but wish to retain compatibility with files saved using the old class name, you can substitute a different class for a given name by using the `[NSKeyedUnArchiver setClass:forClassName:]` method.

6. If you have properties of a type that doesn't support NSCoding (e.g. a struct), and you wish to code them yourself by applying a conversion function, mark the property as unecodable by changing its ivar name and override the `setWithCoder:` and `encodeWithCoder:` methods (remembering to call the super-implementations of those methods to automatically load and save the other properties of the object). Like this:

        @synthesize uncodableProperty = noencode_uncodableProperty; //non-KVC-compliant name
        
        - (void)setWithCoder:(NSCoder *)coder
        {
            [super setWithCoder:coder];
            self.uncodableProperty = DECODE_VALUE([coder decodeObjectForKey:@"uncodableProperty"]);
        }
        
        - (void)encodeWithCoder:(NSCoder *)coder
        {
            [super encodeWithCoder:coder];
            [coder encodeObject:ENCODE_VALUE(self.newProperty) forKey:@"uncodableProperty"];
        }

7. If you have changed the name of a property, but want to check for the existence of the old key name for backwards compatibility, override the `setWithCoder:` method and add a check for the old property as follows:

        - (void)setWithCoder:(NSCoder *)coder
        {
            [super setWithCoder:coder];
            self.newProperty = [coder objectForKey:@"oldProperty"] ?: self.newProperty;
        }

8. If you have changed the name of a property, but want to load *and save* it using the old key name for backwards compatibility, give the new property a non-KVC-compliant ivar name and override the `setWithCoder:`/`encodeWithCoder:` methods to save and load the property using the old name (remembering to call the super-implementations of those methods to automatically load and save the other properties of the object). Like this:

        @synthesize newProperty = noencode_newProperty; //non-KVC-compliant name
        
        - (void)setWithCoder:(NSCoder *)coder
        {
            [super setWithCoder:coder];
            self.newProperty = [coder objectForKey:@"oldProperty"];
        }
        
        - (void)encodeWithCoder:(NSCoder *)coder
        {
            [super encodeWithCoder:coder];
            [coder encodeObject:self.newProperty forKey:@"oldProperty"];
        }
        
        
Release Notes
--------------

Version 2.2.1

- Added missing <Foundation/Foundation.h> import, required on Xcode 6.x

Version 2.2

- Now supports @dynamic properties, allowing AutoCoding to be used with NSManagedObjects
- Added support for additional integer types
- Fixed unit tests

Version 2.1

- Removed automatic NSCopying implementation (see README for details)
- The +codableProperties method will no longer include properties whose ivars are not KVC compliant, even if they are readwrite. This makes it easier to mark readwrite properties as uncodable without needing to override methods
- The +uncodableProperties method is now deprecated
- dictionaryRepresentation method will now no longer include NSNull entries for nil values (these will simply be omitted from the result instead)
- Now complies with the -Weverything warning level

Version 2.0.3

- Now complies with the -Wextra warning level

Version 2.0.2

- Fixed longstanding issue where uncodableProperties / uncodableKeys was ignored for readonly properties. 

Version 2.0.1

- Fixed bug where AutoCoding's NSCopying implementation was not compatible with properties using copy semantics due to lack of a copyWithZone: implementation.
- Now returns YES for respondsToSelector:@selector(copyWithZone:).

Version 2.0

- AutoCoding now implements the NSSecureCoding protocol automatically
- AutoCoding now detects property types automatically and throws an exception by default if encoded classes to not match the expected type
- Autocoding 2.0 is data-compatible with previous releases, but may require code changes when upgrading.

Version 1.3.1

- Fixed issue with CoreData where AutoCoding's copyWithZone: implementation conflicted with NSManagedObjects
- Due to changes in the automatic NSCopying implementation, calling `[super copyWithZone:]` will no longer work. If you need to do this, override the `copy` method and call `[super copy]` instead
- Added Podspec file

Version 1.3

- AutoCoding no longer attempts to encode virtual properties. Only ivar-backed properties will be encoded
- Added dictionaryRepresentation method for quickly accessing all properties of a class
- The codableKeys and uncodableKeys methods are now class methods, allowing them to be used more easily in factory methods 
- The class-level codable/uncodableKeys methods now only return the properties for the class on which they are called, not all of its superclasses as well
- It is no longer necessary to call [super codableKeys] or [super uncodableKeys] and merge arrays when overriding the codableKeys or uncodableKeys class methods on a subclass (overriding the codableKeys instance method is not recommended)

Version 1.2.1

- writeToFile:atomically: method now returns a BOOL to indicate success
- Changed category file names

Version 1.2

- Read-only properties can now be copied and coded as long as they have a KVC compliant ivar (i.e. one whose name matches the property or the property with the _ prefix)
- initWithCoder: no longer calls [self init], in compliance with Apple docs
- codableKeys method now uses caching for better performance
- Exposed previously private setWithCoder: method used by BaseModel library

Version 1.1.2

- Switched constructor to return new type-safe `instancetype` instead of id, making it easier to use dot-syntax property accessors on loaded instances.

Version 1.1.1

- Read-only properties are now excluded from codableKeys
- Added unit tests

Version 1.1

- Added automatic NSCopying implementation
- writeToFile now obeys the useAuxiliaryFile parameter

Version 1.0

- Initial release
