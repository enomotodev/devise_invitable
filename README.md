# DeviseInvitable
[<img
src="https://badge.fury.io/rb/devise_invitable.svg"/>](http://badge.fury.io/rb
/devise_invitable) [<img
src="https://github.com/scambra/devise_invitable/actions/workflows/ci.yml/badg
e.svg"/>](https://github.com/scambra/devise_invitable/actions/workflows/ci.yml
) [<img
src="https://codeclimate.com/github/scambra/devise_invitable/badges/gpa.svg"/>
](https://codeclimate.com/github/scambra/devise_invitable)

It adds support to [Devise](https://github.com/plataformatec/devise) for
sending invitations by email (it requires to be authenticated) and accept the
invitation setting the password.

## Requirements

The latest version of DeviseInvitable works with Devise >= 4.6.

If you want to use devise_invitable with earlier Devise releases (4.0 <= x <
4.6), use version 1.7.5.

## Installation

Install DeviseInvitable gem:

```shell
gem install devise_invitable
```

Add DeviseInvitable to your Gemfile:

```ruby
gem 'devise_invitable', '~> 2.0.0'
```

### Automatic installation

Run the following generator to add DeviseInvitable’s configuration option in
the Devise configuration file (`config/initializers/devise.rb`):

```shell
rails generate devise_invitable:install
```

When you are done, you are ready to add DeviseInvitable to any of your Devise
models using the following generator:

```shell
rails generate devise_invitable MODEL
```

Replace MODEL by the class name you want to add DeviseInvitable, like `User`,
`Admin`, etc. This will add the `:invitable` flag to your model's Devise
modules. The generator will also create a migration file (if your ORM supports
them).

### Manual installation

Follow the walkthrough for Devise and after it's done, follow this
walkthrough.

#### Devise Configuration
Add `:invitable` to the `devise` call in your model (we’re assuming here you
already have a User model with some Devise modules):

```ruby
class User < ActiveRecord::Base
  devise :database_authenticatable, :confirmable, :invitable
end
```

#### ActiveRecord Migration
Add `t.invitable` to your Devise model migration:

```ruby
create_table :users do
  ...
    ## Invitable
    t.string   :invitation_token
    t.datetime :invitation_created_at
    t.datetime :invitation_sent_at
    t.datetime :invitation_accepted_at
    t.integer  :invitation_limit
    t.integer  :invited_by_id
    t.string   :invited_by_type
  ...
end
add_index :users, :invitation_token, unique: true
```

or for a model that already exists, define a migration to add DeviseInvitable
to your model:

```ruby
def change
    add_column :users, :invitation_token, :string
    add_column :users, :invitation_created_at, :datetime
    add_column :users, :invitation_sent_at, :datetime
    add_column :users, :invitation_accepted_at, :datetime
    add_column :users, :invitation_limit, :integer
    add_column :users, :invited_by_id, :integer
    add_column :users, :invited_by_type, :string
    add_index :users, :invitation_token, unique: true
end
```

If you previously used devise_invitable with a `:limit` on
`:invitation_token`, remove it:

```ruby
def up
  change_column :users, :invitation_token, :string, limit: nil
end

def down
  change_column :users, :invitation_token, :string, limit: 60
end
```

## Mongoid Field Definitions
If you are using Mongoid, define the following fields and indexes within your
invitable model:

```ruby
field :invitation_token, type: String
field :invitation_created_at, type: Time
field :invitation_sent_at, type: Time
field :invitation_accepted_at, type: Time
field :invitation_limit, type: Integer

index( { invitation_token: 1 }, { background: true} )
index( { invitation_by_id: 1 }, { background: true} )
```

You do not need to define a `belongs_to` relationship, as DeviseInvitable does
this on your behalf:
```ruby
belongs_to :invited_by, polymorphic: true
```

Remember to create indexes within the MongoDB database after deploying your
changes.
```shell
rake db:mongoid:create_indexes
```

## Model configuration

DeviseInvitable adds some new configuration options:

*   `invite_for`: The period the generated invitation token is valid. After
this period, the invited resource won't be able to accept the invitation.
When `invite_for` is `0` (the default), the invitation won't expire.


You can set this configuration option in the Devise initializer as follow:

```ruby
# ==> Configuration for :invitable
# The period the generated invitation token is valid.
# After this period, the invited resource won't be able to accept the invitation.
# When invite_for is 0 (the default), the invitation won't expire.
# config.invite_for = 2.weeks
```

or directly as parameters to the `devise` method:

```ruby
devise :database_authenticatable, :confirmable, :invitable, invite_for: 2.weeks
```

*   `invitation_limit`: The number of invitations users can send. The default
value of `nil` means users can send as many invites as they want, there is
no limit for any user, `invitation_limit` column is not used.  A setting
of `0` means they can't send invitations. A setting `n > 0` means they can
send `n` invitations. You can change `invitation_limit` column for some
users so they can send more or less invitations, even with global
`invitation_limit = 0`.

*   `invite_key`: The key to be used to check existing users when sending an
invitation. You can use multiple keys. This value must be a hash with the
invite key as hash keys, and values that respond to the `===` operator
(including procs and regexes). The default value is looking for users by
email and validating with `Devise.email_regexp`.

*   `validate_on_invite`: force a record to be valid before being actually
invited.

*   `resend_invitation`: resend invitation if user with invited status is
invited again. Enabled by default.

*   `invited_by_class_name`: the class name of the inviting model. If this is
`nil`, polymorphic association is used.

*   `invited_by_foreign_key`: the foreign key to the inviting model (only used
if `invited_by_class_name` is set, otherwise `:invited_by_id`)

*   `invited_by_counter_cache`: the column name used for counter_cache column.
If this is `nil` (default value), the `invited_by` association is declared
without `counter_cache`.

*   `allow_insecure_sign_in_after_accept`: automatically sign in the user
after they set a password. Enabled by default.

*   `require_password_on_accepting`: require password when user accepts the
invitation. Enabled by default. Disable if you don't want to ask or
enforce to set password while accepting, because is set when user is
invited or it will be set later.


For more details, see `config/initializers/devise.rb` (after you invoked the
`devise_invitable:install` generator described above).

## Configuring views

All the views are packaged inside the gem. If you'd like to customize the
views, invoke the following generator and it will copy all the views to your
application:

```shell
rails generate devise_invitable:views
```

You can also use the generator to generate scoped views:

```shell
rails generate devise_invitable:views users
```

Then turn scoped views on in `config/initializers/devise.rb`:

```ruby
config.scoped_views = true
```

Please refer to [Devise's README](https://github.com/plataformatec/devise) for
more information about views.

## Configuring controllers

To change the controller's behavior, create a controller that inherits from
`Devise::InvitationsController`. The available methods are: `new`, `create`,
`edit`, and `update`. Refer to the [original controllers
source](https://github.com/scambra/devise_invitable/blob/master/app/controller
s/devise/invitations_controller.rb) before editing any of these actions. Your
controller might now look something like this:

```ruby
class Users::InvitationsController < Devise::InvitationsController
  def update
    if some_condition
      redirect_to root_path
    else
      super
    end
  end
end
```

Now just tell Devise that you want to use your controller, the controller
above is `'users/invitations'`, so our routes.rb would have this line:

```ruby
devise_for :users, controllers: { invitations: 'users/invitations' }
```

be sure that you generate the views and put them into the controller that you
generated, so for this example it would be:

```shell
rails generate devise_invitable:views users
```

To change behaviour of inviting or accepting users, you can simply override
two methods:

```ruby
class Users::InvitationsController < Devise::InvitationsController
  private

    # This is called when creating invitation.
    # It should return an instance of resource class.
    def invite_resource
      # skip sending emails on invite
      super { |user| user.skip_invitation = true }
    end

    # This is called when accepting invitation.
    # It should return an instance of resource class.
    def accept_resource
      resource = resource_class.accept_invitation!(update_resource_params)
      # Report accepting invitation to analytics
      Analytics.report('invite.accept', resource.id)
      resource
    end
end
```

## Strong Parameters

When you customize your own views, you may end up adding new attributes to
forms. Rails 4 moved the parameter sanitization from the model to the
controller, causing DeviseInvitable to handle this concern at the controller
as well. Read about it in [Devise
README](https://github.com/plataformatec/devise#strong-parameters)

There are just two actions in DeviseInvitable that allows any set of
parameters to be passed down to the model, therefore requiring sanitization.
Their names and the permited parameters by default are:

*   `invite` (Devise::InvitationsController#create) - Permits only the
authentication keys (like `email`)
*   `accept_invitation` (Devise::InvitationsController#update) - Permits
`invitation_token` plus `password` and `password_confirmation`.


Here is an example of what your application controller might need to include
in order to add these parameters to the invitation view:

```ruby
before_action :configure_permitted_parameters, if: :devise_controller?

protected

  def configure_permitted_parameters
    devise_parameter_sanitizer.permit(:accept_invitation, keys: [:first_name, :last_name, :phone])
  end
```

Here is an example setting a User's first name, last name, and role for a
custom invitation:

# Configuring the InvitationsController to accept :first_name, :last_name, and :role

```ruby
class Users::InvitationsController < Devise::InvitationsController
  before_action :configure_permitted_parameters

  protected

  # Permit the new params here.
  def configure_permitted_parameters
    devise_parameter_sanitizer.permit(:invite, keys: [:first_name, :last_name, :role])
  end
end
```

# Define your roles in the User model

```ruby
class User < ApplicationRecord
  has_many :models

  enum role: {Role 1 Name: 0, Role 2 Name: 1, Role 3 Name: 2, etc...}
end
```

# In the Invitation view

```ruby
<h2><%= t "devise.invitations.new.header" %></h2>

<%= form_for(resource, as: resource_name, url: invitation_path(resource_name), html: { method: :post }) do |f| %>
  <%= render "devise/shared/error_messages", resource: resource %>
  <% resource.class.invite_key_fields.each do |field| -%>
    <div class="field">
      <%= f.label field %><br />
      <%= f.text_field field %>
    </div>
  <% end %>

  <div class="field">
    <%= f.label :first_name %>
    <%= f.text_field :first_name %>
  </div>

  <div class="field">
    <%= f.label :last_name %>
    <%= f.text_field :last_name %>
  </div>

  <div class="field">
    <%= f.label :role %>
    <%= f.select :role, options_for_select(User.roles.map { |key, value| [key.humanize, key] }), {prompt: "Select Role"} %>
  </div>

  <div class="actions">
    <%= f.submit t("devise.invitations.new.submit_button") %>
  </div>
<% end %>
```

## Usage

### Send an invitation

To send an invitation to a user, use the `invite!` class method. **Note: This
will create a user, and send an email for the invite.** `:email` must be
present in the parameters hash. You can also include other attributes in the
hash. The record will not be validated.

```ruby
User.invite!(email: 'new_user@example.com', name: 'John Doe')
# => an invitation email will be sent to new_user@example.com
```

If you want to create the invitation but not send it, you can set
`skip_invitation` to `true`.

```ruby
user = User.invite!(email: 'new_user@example.com', name: 'John Doe') do |u|
  u.skip_invitation = true
end
# => the record will be created, but the invitation email will not be sent
```

When generating the `accept_user_invitation_url` yourself, you must use the
`raw_invitation_token`. This value is temporarily available when you invite a
user and will be decrypted when received.

```ruby
accept_user_invitation_url(invitation_token: user.raw_invitation_token)
```

When `skip_invitation` is used, you must also then set the
`invitation_sent_at` field when the user is sent their token. Failure to do so
will yield "Invalid invitation token" error when the user attempts to accept
the invite. You can set the column, or call `deliver_invitation` to send the
invitation and set the column:

```ruby
user.deliver_invitation
```

You can add `:skip_invitation` to attributes hash if `skip_invitation` is
added to `attr_accessible`.

```ruby
User.invite!(email: 'new_user@example.com', name: 'John Doe', skip_invitation: true)
# => the record will be created, but the invitation email will not be sent
```

`skip_invitation` skips sending the email, but sets `invitation_token`, so
`invited_to_sign_up?` on the resulting user returns `true`.

To check if a particular user is created by invitation, irrespective to state
of invitation one can use `created_by_invite?`

**Warning**

When using `skip_invitation` you must send the email with the user object
instance that generated the tokens, as `user.raw_invitation_token` is
available only to the instance and is not persisted in the database.

You can also set `invited_by` when using the `invite!` class method:

```ruby
User.invite!({ email: 'new_user@example.com' }, current_user) # current_user will be set as invited_by
```

### Sending an invitation after user creation

You can send an invitation to an existing user if your workflow creates them
separately:

```ruby
user = User.find(42)
user.invite!(current_user)  # current user is optional to set the invited_by attribute
```

### Find by invitation token

To find by invitation token use the `find_by_invitation_token` class method.

```ruby
user = User.find_by_invitation_token(params[:invitation_token], true)
```

### Accept an invitation

To accept an invitation with a token use the `accept_invitation!` class
method. `:invitation_token` must be present in the parameters hash. You can
also include other attributes in the hash.

```ruby
User.accept_invitation!(invitation_token: params[:invitation_token], password: 'ad97nwj3o2', name: 'John Doe')
```

### Callbacks

A callback event is fired before and after an invitation is created
(User#invite!) or accepted (User#accept_invitation!). For example, in your
resource model you can add:

```ruby
# Note: callbacks should be placed after devise: :invitable is specified.
before_invitation_created :email_admins
after_invitation_accepted :email_invited_by

def email_admins
  # ...
end

def email_invited_by
  # ...
end
```

The callbacks support all options and arguments available to the standard
callbacks provided by ActiveRecord.

### Scopes

A pair of scopes to find those users that have accepted, and those that have
not accepted, invitations are defined:

```ruby
User.invitation_accepted     # => returns all Users for whom the invitation_accepted_at attribute is not nil
User.invitation_not_accepted # => returns all Users for whom the invitation_accepted_at attribute is nil
User.created_by_invite       # => returns all Users who are created by invitations, irrespective to invitation status
```

## Integration in a Rails application

Since the invitations controller takes care of all the creation/acceptation of
an invitation, in most cases you wouldn't call the `invite!` and
`accept_invitation!` methods directly. Instead, in your views, put a link to
`new_user_invitation_path` or `new_invitation_path(:user)` or even
`/users/invitation/new` to prepare and send an invitation (to a user in this
example).

After an invitation is created and sent, the inviter will be redirected to
`after_invite_path_for(inviter, invitee)`, which is the same path as
`signed_in_root_path` by default.

After an invitation is accepted, the invitee will be redirected to
`after_accept_path_for(resource)`, which is the same path as
`signed_in_root_path` by default. If you want to override the path, override
invitations controller and define `after_accept_path_for` method. This is
useful in the common case that a user is invited to a specific location in
your application. More on [Devise's
README](https://github.com/plataformatec/devise), "Controller filters and
helpers" section.

The invitation email includes a link to accept the invitation that looks like
this: `/users/invitation/accept?invitation_token=abcd123`. When clicked, the
invited must set a password in order to accept its invitation. Note that if
the `invitation_token` is not present or not valid, the invited is redirected
to `invalid_token_path_for(resource_name)`, which by default is
`after_sign_out_path_for(resource_name)`.

The controller sets the `invited_by_id` attribute for the new user to the
current user.  This will let you easily keep track of who invited whom.

## Controller filter

InvitationsController uses `authenticate_inviter!` filter to restrict who can
send invitations. You can override this method in your
`ApplicationController`.

Default behavior requires authentication of the same resource as the invited
one. For example, if your model `User` is invitable, it will allow all
authenticated users to send invitations to other users.

You would have a `User` model which is configured as invitable and an `Admin`
model which is not. If you want to allow only admins to send invitations,
simply overwrite the `authenticate_inviter!` method as follow:

```ruby
class ApplicationController < ActionController::Base
  protected

    def authenticate_inviter!
      authenticate_admin!(force: true)
    end
end
```

And include `DeviseInvitable::Inviter` module into `Admin` model:

```ruby
class Admin < ActiveRecord::Base
  devise :database_authenticatable, :validatable
  include DeviseInvitable::Inviter
end
```

## Has many invitations

If you want to get all records invited by a resource, you should define
`has_many` association in the model allowed to send invitations.

For the default behavior, define it like this:

```ruby
has_many :invitations, class_name: self.to_s, as: :invited_by
```

For the previous example, where admins send invitations to users, define it
like this:

```ruby
has_many :invitations, class_name: 'User', as: :invited_by
```

## I18n

DeviseInvitable uses flash messages with I18n with the flash keys
`:send_instructions`, `:invitation_token_invalid` and `:updated`. To customize
your app, you can modify the generated locale file:

```yaml
en:
  devise:
    invitations:
      send_instructions: 'An invitation email has been sent to %{email}.'
      invitation_token_invalid: 'The invitation token provided is not valid!'
      updated: 'Your password was set successfully. You are now signed in.'
      updated_not_active: 'Your password was set successfully.'
```

You can also create distinct messages based on the resource you've configured
using the singular name given in routes:

```yaml
en:
  devise:
    invitations:
      user:
        send_instructions: 'A new user invitation has been sent to %{email}.'
        invitation_token_invalid: 'Your invitation token is not valid!'
        updated: 'Welcome on board! You are now signed in.'
        updated_not_active: 'Welcome on board! Sign in to continue.'
```

The DeviseInvitable mailer uses the same pattern as Devise to create mail
subject messages:

```yaml
en:
  devise:
    mailer:
      invitation_instructions:
        subject: 'You got an invitation!'
        user_subject: 'You got a user invitation!'
```

Take a look at the [generated locale
file](https://github.com/scambra/devise_invitable/blob/master/config/locales/e
n.yml) to check all available messages.

Check out [wiki](https://github.com/scambra/devise_invitable/wiki/I18n) for
translations.

### Use with sub schema
If you are using sub schema in you application, you need to make sure that you
are prioritizing your sub schema scheme over Warden in Rack. For instance, if
you are using the Apartment gem go inside your `config/application.rb` file,
add the following lines:

```ruby
module YourSite
  class Application < Rails::Application
    ...
    Rails.application.config.middleware.insert_before Warden::Manager, Apartment::Elevators::Subdomain
  end
end
```


## Other ORMs

DeviseInvitable supports ActiveRecord and Mongoid, like Devise.

## Wiki

It's possible to find additional information about DeviseInvitable on the
Wiki:

https://github.com/scambra/devise_invitable/wiki

## Testing

To run tests:

```shell
bundle install
bundle exec rake test
```

## Contributors

Check them all at:

https://github.com/scambra/devise_invitable/contributors

Special thanks to [rymai](https://github.com/rymai) for the Rails 3 support,
his fork was a great help.

## Note on Patches/Pull Requests

*   Fork the project.
*   Make your feature addition or bug fix.
*   Add tests for it. This is important so I don't break it in a future
version unintentionally.
*   Commit, do not mess with rakefile, version, or history. (if you want to
have your own version, that is fine but bump version in a commit by itself
I can ignore when I pull)
*   Send me a pull request. Bonus points for topic branches.


## Copyright

Copyright (c) 2019 Sergio Cambra. See LICENSE for details.
