## Refactoring Obese Rails Models
####Joey Lorich


## Fat model - Skinny Controller
  - Great idea in a general sense
  - Keep most logic out of the controller
  - Commonly misunderstood


## So what's the problem?
  - Rails often encourages you to just put everything in the ActiveRecord::Base model files
  - Eventually your model files are unmaintainably 'fat'


## Fat controller method that needs refactoring

    # Add a family member POST action
    def add_family_member
      authorize! :update, @current_user

      @user = @current_user
      @active_profile_link = 'family'

      @email = params[:email]
      @email_confirmation = params[:email_confirmation]
      
      # limit to 2 adults
      if @current_user.family.adults.length >= 2
        flash[:error] = "There may only be 2 adults per family."
        return render :add_family_member
      end

      # Validate email
      return render :add_family_member unless validate_emails(@email, @email_confirmation)

      # Find a new user
      invited_user = User.find_or_initialize_by(email:@email)

      # Make a shell user if no user exists
      if invited_user.new_record?
        invited_user.skip_confirmation!
        invited_user.invitation_token = SecureRandom.hex(15)
        invited_user.save(validate: false)

      end

      # Find or create an invitation
      invitation = FamilyInvite.find_or_initialize_by(
        family_id: @current_user.family.id,
        user_id: @user.id,
        invited_user_id: invited_user.id,
        rejected: false,
        accepted: false
      )

      # Only allow invites every X days
      if !invitation.new_record?
        flash.now[:error] = "This person already has a pending invitation to join your family."
        return render :add_family_member
      end
      
      # Create invitation record
      # This also makes appropriate UserAlerts after_create
      invitation.save

      # Send email invitation
      InviteMailer.send_family_invite(invited_user, @current_user, params["message"].to_s).deliver

      # Flash success
      flash.now[:notice] = "#{params[:email]} has been invited to your family."

      # Clear email informaiton
      @email = @email_confirmation = ""

      return render :add_family_member
    end


## How it's commonly "fixed"

	class Family < ActiveRecord::Base
	  def invite(email, inviting_user)
	     # Find a new user
	    invited_user = User.find_or_initialize_by(email, inviting_user)
	
	    # Make a shell user if no user exists
	    if invited_user.new_record?
	      invited_user.skip_confirmation!
	      invited_user.invitation_token = SecureRandom.hex(15)
	      invited_user.save(validate: false)
	    end
	
	    # Find or create an invitation
	    invitation = FamilyInvite.find_or_initialize_by(
	      family_id: self.id,
	      user_id: inviting_user.id,
	      invited_user_id: invited_user.id,
	      rejected: false,
	      accepted: false
	    )
	
	    # Only allow invites every X days
	    if !invitation.new_record?
	      return "This person already has a pending invitation to join your family."
	    end
	    
	    # Create invitation record
	    # This also makes appropriate UserAlerts after_create
	    invitation.save
	
	    # Send email invitation
	    InviteMailer.send_family_invite(invited_user, @current_user, params["message"].to_s).deliver
	
	    return true
	  end
	end


## "Skinny Controller"

    class FamilyController < ActionController::Base
	  def add_family_member
	    authorize! :update, current_user
	
	    @email = params[:email]
	    @email_confirmation = params[:email_confirmation]
	
	    # limit to 2 adults
	    if @current_user.family.adults.length >= 2
	      flash[:error] = "There may only be 2 adults per family."
	      return render :add_family_member
	    end
	
	    # Validate email
	    return render :add_family_member unless validate_emails(params[:email], params[:email_confirmation])
	
	    invitation_result = @current_user.family.invite(params['email'], current_user)
	
	    if (invitation_result == true)
	      flash.now[:notice] = "#{params[:email]} has been invited to your family."
	      @email = @email_confirmation = ""
	    else
	      flash.now[:error] = invitaton_result.
	    else
	
	    return render :add_family_member
	  end
	end


## Problems
 - Clutters the family model
 - A Family doesn't really have anything directly to do with inviting new users, creating shell users, sending out mail.
 - The Family class is doing too much


## What really is the 'model'
  - Model layer, not model file
  - Not just ActiveRecord::Base subclasses
  - Data representation as well as business logic


## Obese Model Files
 - Hundreds or even thousands of lines of code
 - Hard to navigate
 - Too large to effectively organize
 - Massive test files
 - Often not very independent (easy to just use lots of shared ivars)


## Classes should be SOLID, not fat
  - Single responsibility principle
    - A class should have only a single responsibility.
  - Open/closed principle
    - Software entities should be open for extension, but closed for modification
  - Liskov substitution principle
    - Objects in a program should be replaceable with instances of their subtypes without altering the correctness of that program (See design by contract)
  - Interface segregation principle
    - No client should be forced to implement methods it does not need
  - Dependency inversion principle
    - One should depend upon abstractions, not concretion (See Dependency Injection)


## Obese models are not SOLID
  - Breaking the single responsibility principle
  - Often highly dependent on other code
  - Too many specific features to be reasonably extended
  - Too many specific features to be substituted


## Some options for slimming down
  - Concerns
  - Service Objects
  - Query Objects
  - Policy Objects
  - Decorators


## Concerns
 - Essentially a mixin, but with a few helpers to simplify things
 - A simple way to pull out shared code into a module


## Example
    module Taggable
      extend ActiveSupport::Concern

      included do
        has_many :taggings, as: :taggable
        has_many :tags, through: :taggings
      end

      def tags_string
        tags.map(&:name).join(', ')
      end

      def tags_string=(tag_string)
        tag_names = tag_string.to_s.split(', ')

        tag_names.each do |tag_name|
          tags.build(name: tag_name)
        end
      end
    end


## Problems
 - Often simply moving code instead of logically abstracting out
 - Can make relationships / method implementations non-obvious
 - Easily abusable


## How to use concerns
 - Don't use them to simply move methods out
 - Make sure it's code that can be sensibly reused


## Service Objects
 - A separate ruby class to help handle some kind of interaction
 - Should be relatively isolated and reusable


## When to use service objects
 - Complex actions
 - Cross-model actions
 - Interfacing with a third party service
 - When there are multiple ways for performing the same action (e.g. authentication)


## FamilyInvitationService
    class FamilyInvitationService
      attr_accessor :errors

      def initialize(family)
        @family = family
        @errors = []
      end

      def invite_to_family(email, email_confirmation, inviting_user, message)
        return false if maximum_number_of_adults_reached?
        return false if !valid_email?(email, email_confirmation)

        invited_user = User.find_by(email: email) || create_shell_user

        return false unless generate_invitation(invited_user, inviting_user)

        InviteMailer.send_family_invite(
          invited_user,
          inviting_user,
          message
        ).deliver

      end

      def create_shell_user(email)
        shell_user = User.new(email: email)
        shell_user.skip_confirmation!
        invited_user.invitation_token = SecureRandom.hex(15)
        shell_user.save(validate: false)
      end

      def generate_invitation(invited_user, inviting_user)
        invite = FamilyInvite.find_or_initialize_by(
          family_id: @family
          user_id: inviting_user.id,
          invited_user_id: invited_user.id,
          rejected: false,
          accepted: false
        )

        unless invite.new_record?
          @errors << "This person already has a pending invitation to join your family."
          return false
        end

        invite.save
      end

      def maximum_number_of_adults_reached?
        return false unless adults.length < 2

        @errors << "There may only be 2 adults per family."
        true
      end

      def valid_email?(email, email_confirmation)
        return true if EmailValidationService.new(params[:email], params[:email_confirmation]).validate

        @errors << "The email entered is not valid"
      end
    end


## Refactored controller

    def add_family_member
      authorize! :update, @current_user
      @active_profile_link = 'family'

      familyInvitationService = FamilyInvitationService.new(user.family)

      unless familyInvitationService.invite_to_family(
          params[:email],
          params[:confirmation],
          @current_user,
          params[:message]
        )

        flash[:error] = familyInvitationService.errors.join("\n")
        return render :add_family_member
      end

      flash.now[:notice] = "#{params[:email]} has been invited to your family."

      @email = @email_confirmation = ""

      return render :add_family_member
    end


## Benefits
 - Less responsibilities per class 
 - More easily testable
 - More Reusable
 - Keeps third party services/notifications/mailers out of the models


## Query Objects
 - Extract complex queries or reused common queries into ruby objects


## Example
    class Child < ActiveRecord::Base
      def inclusive_commitments_in_for(activity)
        # own_commitment = Commitment.where("activity_id = ? AND child_id = ? AND going = ?", activity.id, id, true).first
        commitments = friends.includes(:commitments).where("commitments.activity_id = ? AND commitments.going = ?", activity.id, true)
        commitments.unshift(self) if is_going_to?(activity)
        commitments
      end

      def inclusive_commitments_interested_for(activity)
        # own_commitment = Commitment.where("activity_id = ? AND child_id = ? AND interested = ?", activity.id, id, true).first
        commitments = friends.includes(:commitments).where("commitments.activity_id = ? AND commitments.interested = ?", activity.id, true)
        commitments.unshift(self) if is_interested_in?(activity)
        commitments
      end

      def inclusive_commitments_out_for(activity)
        # own_commitment = Commitment.where("activity_id = ? AND child_id = ? AND not_going = ?", activity.id, id, true).first
        commitments = friends.includes(:commitments).where("commitments.activity_id = ? AND commitments.not_going = ?", activity.id, true)
        commitments.unshift(self) if is_not_going_to?(activity)
        commitments
      end

      def is_going_to?(activity)
        !Commitment.where("activity_id = ? AND going = ? AND child_id = ?", activity.id, true, id).blank?
      end

      def is_interested_in?(activity)
        !Commitment.where("activity_id = ? AND interested = ? AND child_id = ?", activity.id, true, id).blank?
      end

      def is_not_going_to?(activity)
        !Commitment.where("activity_id = ? AND not_going = ? AND child_id = ?", activity.id, true, id).blank?
      end
    end


## Example
    class ChildCommitmentsQuery
      def initialize(child)
        @child = child
      end
      
      def is_going_to?(activity)
        commitment_exists(activity, :going)
      end

      def is_interested_in?(activity)
        commitment_exists(activity, :interested)
      end

      def is_not_going_to?(activity)
        commitment_exists(activity, :not_going)
      end
     
      def inclusive_commitments_in_for(activity)
        inclusive_commitments_for(activity, :going)

      def inclusive_commitments_interested_for(activity)
        inclusive_commitments_for(activity, :interested)
      end

      def inclusive_commitments_out_for(activity)
        inclusive_commitments_for(activity, :not_going)
      end

      private

      def commitment_exists?(activity, status)
        Commitment.where(
          activity_id: activity.id,
          child_id: @child.id,
          status.to_sym => true
        ).present?
      end

      def commitments_for(activity)
        friends.includes(:commitments).where(
          commitments: {activity_id: activity.id}
        )
      end

      def commitments_for(activity, status)
        commitments_for(activtiy).where(
          commitments: {status.to_sym => true}
        )
      end

      def inclusive_commitments_for(activity, status)
        commitments = commitments_for(activity, status)
        commitments.unshift(@child) if commitment_exists?(activity, status)
        commitments
    end


## Policy Objects
  - Contains read-only methods 
  - Comparisons
  - Roles checking (though we should use cancan for what we can)


## Example
    class UserPolicy
      def initialize(user)
        @user = user
      end

      def is_default?
        @user == User.default
      end
      
      def is_test_drive?
        return self.roles.include?(Role.where(name: 'Test Drive').first)
      end

      def is_organization?
        @user.class == OrganizationUser
      end
    end


## Decorators
 - Work as a cleaner alternative to callbacks
 - Classes that wrap other functionality
 - "Decorators differ from Service Objects because they layer on responsibilities to existing interfaces. Once decorated, collaborators just treat the Decorator instance as if it were the object its decorating"


## Example
    class UserUpdateNotifier
      def initialize(user)
        @user = user
      end

      def save
        @user.save && @user.notify_if_changed && @user
      end
 
      private

      def notify_if_changed
        if @user.changed?
          @user.notify
        emd
      end

      def notify(notification: nil, key: 'kUWUserUpdated', alert: nil)
        if !@user.devices.urban_airship.blank?
          notification ||= {
            device_tokens: @user.devices.urban_airship.collect {|d| d.token },
            aps: {key: key}
          }
          
          notification[:aps][:alert] = alert if alert
          notification[:aps][:badge] = @user.badge_count

          Urbanairship.push(notification)
        end
      end
    end

    #usage
	user.assign_attributes(user_params)
    UserUpdateNotifier.new(user).save


## Too many classes?
  - No
  - Stick with good design and you'll have more flexible and more easily maintainable code - meaning you will be *faster* overall


## Still seem disorganized? Namespaces!
  - Use namespaces!
  - There's no need for one models folder with 50+ models.  It's confusing and slows you down.
  - Namespaces promote organization, isolation, and make finding things easier


## Inspired by
- [7 Patterns to Refactor Fat ActiveRecord Models](http://blog.codeclimate.com/blog/2012/10/17/7-ways-to-decompose-fat-activerecord-models/)
- [Practical Object-Oriented Design in Ruby - Sandi Metz](http://www.poodr.com/)