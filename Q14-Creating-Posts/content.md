---
title: "Improving the Upload Code and Adding a Post Class"
slug: improving-photo-upload-firebase
---

Now it's time to move from a working solution to a good one. We need to store more information along with the `Post` that we're creating. Right now we are only storing the image file, but we also need to store the `User` to which the post belongs. This means that we'll need to create a `Post` class.

First let's start by creating a post in our database everytime a new image is uploaded to Firebase. Create a new class `Post.swift`:

    import UIKit
    import FirebaseDatabase.FIRDataSnapshot

    class Post {
        // properties and initializers
    }

Next let's add properties to store all the additional information we need. Add the following to your post class.

    class Post {
        var key: String?
        let imageURL: String
        let imageHeight: CGFloat
        let creationDate: Date
    }
    
You'll get some compiler errors for not having an initialiers or default values for certain properties. Let's go ahead and fix that:

    init(imageURL: String, imageHeight: CGFloat) {
        self.imageURL = imageURL
        self.imageHeight = imageHeight
        self.creationDate = Date()
    }
    
Here we create an initializer to take an image URL and image height and creates a `Post` object.

# Create a New Post

After getting download URL from our `PostService`, we then we to create a new `Post` object that gets stored in the database. First let's create a new service method in `PostService`. This time, our class method will be private, because we only want our `PostService.create(for:)` method to be able to create a `Post` in the database once we have uploaded the image to Firebase storage. Add the following method definition to your `PostService`:

    private static func create(forURLString urlString: String, aspectHeight: CGFloat) {
        // create new post in database
    }

Here we'll setup the network code to create a new Post JSON object in our Firebase database. To do this, we'll need to add the following code in our create(forURLString:aspectHeight:) method:

    let currentUser = User.current
    let post = Post(imageURL: urlString, imageHeight: aspectHeight)
    let dict = post.dictValue

    let postRef = FIRDatabase.database().reference().child("posts").child(currentUser.uid).childByAutoId()
    postRef.updateChildValues(dict)
    
This will create a JSON object in our database. Let's take a look at what we do:

First we get reference to our current user. We will use our user's uid to assign the post to a user. Next we create a new `Post` object in memory with an image URL and an aspect height. Then we need to convert this value to a dictionary so we can store it in our database. To turn our post into a dictionary of type `[String : Any]` we'll need to add the following computed variable to our `Post` class:

    var dictValue: [String : Any] {
        let createdAgo = creationDate.timeIntervalSince1970
        
        return ["image_url" : imageURL,
                "image_height" : imageHeight,
                "created_at" : createdAgo]
    }
    
After we have our dictionary, we then create a path to the location of where we'll store our post JSON. Initially you might think of storing our `Post` as part of our `User`, however we'll so that our JSON tree isn't nested. To do so, we'll break out `Post` into it's own subtree. To tell which user posted the `Post`, we'll separate each `Post` first by each user's `uid`. This way, if we were to read our `User` from our database, we wouldn't have to fetch all of the user's posts with it. This is especially good if each user has many posts.

Last thing we'll need to do to hook up this method is to call our `create(forURLString:aspectHeight:)` from our `create(for:)` method once we have an image URL. Go ahead and insert the following in `create(for:)`:

    let urlString = downloadURL.absoluteString
    create(forURLString: urlString, aspectHeight: 320)

You'll notice here that we hardcode the aspect height of the image. The reason we want to store the aspect height is because when we render our image, we'll need to know the height of the image to display. We do this by calculating what the height of the image should be based on the maximum width and height of an iPhone. 

We'll create a new image extension that calculates the aspect height based on the size of the iPhone 7 plus. Create a new file under extensions called:

    import UIKit

    extension UIImage {
        var aspectHeight: CGFloat {
            let heightRatio = size.height / 736
            let widthRatio = size.width / 414
            let aspectRatio = fmax(heightRatio, widthRatio)

            return size.height / aspectRatio
        }
    }

We added a computed property to UIImage that will calculate the aspect height for the instance of a UIImage based on the size property of an image. Now we can update our `create(for:)` method in our `PostService`:

    static func create(for image: UIImage) {
        let imageRef = FIRStorage.storage().reference().child("test_image.jpg")
        StorageService.uploadImage(image, at: imageRef) { (downloadURL) in
            guard let downloadURL = downloadURL else {
                return
            }
            
            let urlString = downloadURL.absoluteString
            let aspectHeight = image.aspectHeight
            create(forURLString: urlString, aspectHeight: aspectHeight)
        }
    }
    
Last, we'll need to create a more suitable location for our post image files to be stored. Right now, since we're storing all of our images at the same path, they're being overwritten. Let's create a new extension for `FIRStorageReference` that generates a new storage location for each user's post:

Create a new extension file called `FIRStorageReference+Post.swift` with the following content:

    import Foundation
    import FirebaseStorage

    extension FIRStorageReference {
        static let dateFormatter = ISO8601DateFormatter()

        static func newPostImageReference() -> FIRStorageReference {
            let uid = User.current.uid
            let timestamp = dateFormatter.string(from: Date())

            return FIRStorage.storage().reference().child("images/posts/\(uid)/\(timestamp).jpg")
        }
    }

Here we create an extension to FIRStorageReference with a class method that will generate a new location for each new post image that is created by the current ISO timestamp.

We can update our code to use the new location generation:

    static func create(for image: UIImage) {
        let imageRef = FIRStorageReference.newPostImageReference()
        StorageService.uploadImage(image, at: imageRef) { (downloadURL) in
            guard let downloadURL = downloadURL else {
                return
            }
            
            let urlString = downloadURL.absoluteString
            let aspectHeight = image.aspectHeight
            create(forURLString: urlString, aspectHeight: aspectHeight)
        }
    }


Next let's delete the test posts we created in our database and storage so we don't have any incomplete data. Then let's go ahead and run the app and test out our code. If everything works as expected, we'll see that a new `Post` object is created in our database.

# Conclusion

We've improved the photo upload mechanism a lot! We've created our own custom `Post` class and added a class method in our `PostService` to create a `Post` JSON object in our database.

In our next step we'll look at reading data from our database and displaying it to our users.