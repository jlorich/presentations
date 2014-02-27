## Refactoring Obese Models
####Joey Lorich


## Fat model - Skinny Controller
  - Great idea in a general sense
  - Keep most logic out of the contorller
  - However, models quickly become unmaintainably 'fat'


## Obese Models
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
    - No client should be forced to depend on methods it does not use
  - Dependency inversion principle
    - One should “Depend upon Abstractions. Do not depend upon concretion (See Dependency Injection)


## Obese models are not SOLID
  - Breaking the single responsibility principle all over
  - Often highly dependent on other code
  - Too many specific features to be extended
  - Too many specific features to be substituted


## Options for slimming down
  - Concerns
  - Service Objects
  - Form Objects
  - Query Objects
  - View Objects
  - Policy Objects
  - Decorators


## Concerns
 - Essentially a mixin, but with a little rails magic to make sure you can't do a few things
 - A simple way to pull out shared code into an additional class


## DHH Example
    module Taggable
      extend ActiveSupport::Concern

      included do
        has_many :taggings, as: :taggable
        has_many :tags, through: :taggings

        class_attribute :tag_limit
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

      module ClassMethods

        def tag_limit(value)
          self.tag_limit_value = value
        end

      end
    end


## Uwithus Example
    module Personable
      extend ActiveSupport::Concern

      def full_name
        "#{first_name.to_s.capitalize} #{last_name.present? ? last_name.to_s.capitalize : family.name.to_s.capitalize}"
      end

      def member_of? (family_or_organization)
        case family_or_organization
          when Family
            self.family == family_or_organization
            
          when Organization
            if self.class == OrganizationUser
              family_or_organization.organization_users.include?(self)
            else
              false
            end
          end
      end

      def transfer_to_family(family)
        raise "No destination family provided" if family.blank?
        
        previous_family = self.family

        self.family = family

        if previous_family && previous_family.adults.blank? && previous_family.children.blank?
          previous_family.destroy
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
 - A seperate basic ruby class to help handle some kind of interaction
 - Should be relatively isolated and reusable
 - Can provide a good alternative to after_xxxx hooks


## When to user service objects
 - Complex actions
 - Cross-model actions
 - Interfacing with a third party service
 - When there are multiple ways for performing the same action (e.g. authentication)


## Method that needs refactoring
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


## FamilyInvitationService
    class FamilyInvitationService
      attr_accessor :errors

      def initialize(family)
        @family = family
        @errors = []
      end

      def invite_to_family(email, email_confirmation, inviting_user, message)
        return false if maximum_number_of_adults_reached?
        return false if valid_email?(email, email_confirmation)

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

      # Flash success
      flash.now[:notice] = "#{params[:email]} has been invited to your family."

      # Clear email informaiton
      @email = @email_confirmation = ""

      return render :add_family_member
    end


## Benefits
 - Easily testable
 - Reusable
 - Keeps third party services/notifications/mailers out of the models


## Form Objects
 - A class represending incoming data
 - A cleaner alternative to nested attributes


## Current Setup
    class Activitycard < ActiveRecord::Base
      # do not change the order of these accepts_nested_attributes_for calls
      # if the order changes, callbacks will get screwed up
      belongs_to :inspiration, :dependent => :destroy
      accepts_nested_attributes_for :inspiration, :allow_destroy => false
      
      belongs_to :activity_permission
      belongs_to :owner, :polymorphic => true
      belongs_to :detail, :polymorphic => true, :dependent => :destroy

      has_one :activity_detail, :inverse_of => :activity_card
      accepts_nested_attributes_for :activity_detail, :allow_destroy => false

      has_one :organization_activity_detail, :inverse_of => :activity_card
      accepts_nested_attributes_for :organization_activity_detail, :allow_destroy => false
      
      has_many :commitments, :dependent => :destroy, :inverse_of => :activity_card
      accepts_nested_attributes_for :commitments, :allow_destroy => true
      has_many :commitment_visibilities, :through => :commitments

      has_many :user_alerts, :dependent => :destroy

      has_many :uvites, :dependent => :destroy
      has_many :uvite_invitations, :dependent => :destroy

      has_many :user_viewers, :through => :commitment_visibilities, :source => :owner, :source_type => "User"
      has_many :organization_viewers, :through => :commitment_visibilities, :source => :owner, :source_type => "Organization"
        
      has_many :events, :dependent => :destroy

      has_many :activity_card_sub_categories, :dependent => :destroy
      accepts_nested_attributes_for :activity_card_sub_categories, :allow_destroy => true

      has_many :sub_categories, through: :activity_card_sub_categories
      accepts_nested_attributes_for :sub_categories, :allow_destroy => true
      
      belongs_to :category


## Example
    class ActivityCardCreate
      include Virtus # Adds in some activerecord-like functionliaty (https://github.com/solnic/virtus)

      extend ActiveModel::Naming
      include ActiveModel::Conversion
      include ActiveModel::Validations

      attr_reader :user, :company, :inspiration, :commitment

      attribute :name, String
      attribute :description, String
      attribute :image_path, String
      attribute :commitment_status, String
      
      validates :name, presence: true
      validates :description, presence: true

      # Forms are never themselves persisted
      def persisted?
        false
      end

      def save
        if valid?
          persist!
          true
        else
          false
        end
      end

    private

      def persist!
        @card = ActivityCard.create!()
        @detail = ActivityDetail.create!(name: company_name)
        @inspiration = Inspiration.create(image_path: image_path)
        @commitment = Commitment.create(activity_card: @card, user: current_user)
      end
    end


## Query Objects
 - Extract complex queries or reused commin queries into ruby objects


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


## View Objects (ViewModels)
 - Objects that act as a 'contract' between controller and view
 - Contain all data that should be needed to display a model
 - Contain *only* data needed to display a model
 - Somewhat like a Seralizer object from ActiveModelSerializers


##Example
    class ChildView
      attr_accessor child

      def initialize(child)
        @child = child
      end

      def possessive_pronoun
        @child.gender == "boy" ? "his" : "her"
      end

      def badge_avatar
        if @child.avatar.respond_to?(:expiring_url)
          (return @child.avatar.expiring_url(1.minute, :badge)) unless @child.avatar_file_name.blank?
          "/assets/content/profile-blank-example.png"
        else
          (return @child.avatar.url(:badge)) unless @child.avatar_file_name.blank?
          "/assets/content/profile-blank-example.png"
        end
      end
    end


## Policy Objects
  - Contains read-only methods 
  - Analytics information
  - Comparisons
  - Roles checking (though we should use cancan for what we can)


## Example
    class UserPolicy
      def initialize(user)
        @user = user
      end

      def is_oliver?
        @user == User.oliver
      end
      
      def is_test_drive?
        return self.roles.include?(Role.where("name = ?", "Test Drive").first)
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
        @user.save && @user.notify_if_changed
      end

      def changed?
        @user.changed?
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
    UserUpdateNotifier.new(
      user.assign_attributes(user_params)
    ).save


### Well-written decorators are nestable
    UserChangedPushNotifier.new(
      UserChangedSocketNotifier.new(
        user.assign_attributes(user_params)
      )
    ).save


## Too many classes?
  - Probably not.
  - Stick with good design and you'll have more flexible and more easily maintainable code - meaning you will be *faster* overall


## Still seem disorganized? Namespaces!
  - Use namespaces!
  - There's no need for one models folder with 50+ models.  It's confusing and slows you down.
  - Namespaces promote organization, isolation, and make finding things easier


## Source
- Inspred by http://blog.codeclimate.com/blog/2012/10/17/7-ways-to-decompose-fat-activerecord-models/