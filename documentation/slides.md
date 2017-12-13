## Software Documentation
####Joey Lorich


##Overview
 - What is documentation?
 - YARD
 - Proposed Standards


## What is software documentation?
> "Written text that accompanies computer software. It either explains how it operates or how to use it, and may mean different things to people in different roles." - *Wikipedia (I'm lazy)*


## Why Document Source Code?
 - It saves money.  
   - Good code *should* be fairly self explanatory, but it's not always.
   - Just because you *can* figure out what something does doesn't mean that it doesn't waste time a quick comment could have saved.
   - Wasted time means less efficient development and wasted client money


## Still not convinced?
 - You won't be the only developer ever on a project
 - Even if you think you might be, you probably won't. Act accordingly
 - Sometimes you can write beautiful, fancy, concise code that's difficult for thers to understand.
 - Sometimes that code is difficult for you to understand two months from now


## Still not convinced?
 - It helps new developers learn about the project faster
 - It encourages organization, which speeds future development


## So what does documentation mean to Cloudspace?
As software developers there are four categories of documentation we tend to run across with work.


## Types of Documentation
 - User Documentation
 - Design Documentation
 - Inline Documentation
 - Formal Source Documentation


## User Documentation
 - Documentation specifically written for the end user.
 - Manuals, usage instructions, FAQs, etc.
 - We don't really write much of this at Cloudspace.


## Design Documentation
 - Entity-Relationship Diagrams, Workflows, etc
 - A birds-eye view of how the system goes together

<img class="fragment" src="http://i.imgur.com/1LmyxQt.jpg" style="height: 400px" />


## Inline Documentation
 - Probably what you think of when someone says "documentation"
 - Goes right in the source code (e.g. comments)
 - Explains what and more importantly **why** you're doing something
 - Can be used to generate the Formal Source Documentation


## Formal Source Documentation
 - A separate document containing information about specific pieces of source code (e.g. [http://api.rubyonrails.org/](http://api.rubyonrails.org/))
 - Describes each class, attribute, method, etc
 - Often includes dependency lists, setup instructions, and usage examples


## What's the most efficient way to get all this done?
 - As contracted developers, most general User and Design documentation is given to us by clients - BOOM! done
 - Database ERDs can be generated (see [Rails ERD](http://rails-erd.rubyforge.org/))
 - Formal documentation generally includes information based on source code and there are several tools to help generate it right from the code (see [Rdoc](http://rdoc.sourceforge.net/), [YARD](http://yardoc.org/))


## YARD
###### Yay! Another Ruby Documentation tool


## Why Yard?

- Conveniently generates formal documentation based on code comments
- Full compatible with Rdoc syntax 
- Easy meta-tag formatting like in Python, Java, Objective-C and other language
- Very customizable, with gems available for rails/activerecord support
- Auto-reloading local documentation server


## A basic example: 

    # Reverses the contents of a String or IO object.
    #
    # @param [String, #read] contents the contents to reverse
    # @return [String] the contents reversed lexically
    def reverse(contents)
     contents = contents.read if contents.respond_to? :read
     contents.reverse
    end


## Proposed Standards
 - Use large organizational comments to separate sections of files
 - Document each Class, Attribute, Method, Relationship, and Scope
 - Identify each method parameter and the what it returns (including types)
 - Never introduce a commit that lowers the comment coverage percentage
 - Use [yard-activerecord](https://github.com/theodorton/yard-activerecord)


## What should they look like


## Class comments
 - Explain what the class represents from a business perspective
 - Explain any general business logic related to the class
 - Give examples of usage
 - Detail each attribute inside the class comment


```
# Represents an alert for a user
#
# These alerts are can contain information about many different types of events and classes
#
# @!attribute viewed
#   @return [Boolean] Whether or not this alert has been viewed
#
# @!attribute message
#   @return [Text] A message to go along with this alert
#
# @!attribute created_at
#   @return [DateTime] When this {UserAlert} was created
#
# @!attribute updated_at
#   @return [DateTime] When this {UserAlert} was last updated
class UserAlert < ActiveRecord::Base
  ...
end 
```


## Attribute Comments
 - `yard-activerecord` shows all undocumented attributes automatically
 - Document them in the class comments anyway so all return types and explanations can be specified


## Method comments
 - Explain the business logic behind the method.  This should define the expected behavior
 - Explain what the method does in a general sense
 - Explain common expected interactions with other code


 ```
# Creats a new Connection Request {UserAlert}
#
# Called when a {FamilyConnectionInvite} is created, inviting the invited_user's family to connect
#
# @param [Hash] args Optional arguments for this alert
# @options args [User] :invited_user The user this alert is for
# @options args [Integer] :referenced_user_id The user that created the {FamilyConnectionInvite}
# @options args [User] :friend_invite_id The FamilyConnectionInvite this alert is associated with
#
# @return [Boolean] Whether or not the alert was created
def self.create_connection_request_for(invited_user, args = {})
  create(args.merge({
    user_id: invited_user.id,
    user_alert_category: UserAlertCategory.invitation,     user_alert_message_type: UserAlertMessageType.by_name(:connection_request)
  }))
end
```


## Relationship comments

Simply describe the relationship

    # A {User} this alert my recerence, generally this is the
    # person who initiated the event that caused the alert
    belongs_to :referenced_user, :class_name => "User"


## Scope comments
Scopes are just class methods with a little Rails magic thrown in, document them as such

    # Alerts filtered by category id
    # @param category_id [Integer] The category ID to filter on
    # @return [ActiveRecord::Relation<UserAlert>]
    scope :by_category_id, lambda { |category_id| where("`user_alerts`.`user_alert_category_id` = ?", category_id) }


## Conclusion
Write lots of comments. They make everything better for everyone.
