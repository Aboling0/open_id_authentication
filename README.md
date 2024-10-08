# OpenIdAuthentication

<div id="badges">

[![CI Supported Build][🚎s-wfi]][🚎s-wf]
[![CI Unsupported Build][🚎us-wfi]][🚎us-wf]
[![CI Style Build][🚎st-wfi]][🚎st-wf]
[![CI Coverage Build][🚎cov-wfi]][🚎cov-wf]
[![CI Heads Build][🚎hd-wfi]][🚎hd-wf]

[🚎s-wf]: https://github.com/VitalConnectInc/open_id_authentication2/actions/workflows/supported.yml
[🚎s-wfi]: https://github.com/VitalConnectInc/open_id_authentication2/actions/workflows/supported.yml/badge.svg
[🚎us-wf]: https://github.com/VitalConnectInc/open_id_authentication2/actions/workflows/unsupported.yml
[🚎us-wfi]: https://github.com/VitalConnectInc/open_id_authentication2/actions/workflows/unsupported.yml/badge.svg
[🚎st-wf]: https://github.com/VitalConnectInc/open_id_authentication2/actions/workflows/style.yml
[🚎st-wfi]: https://github.com/VitalConnectInc/open_id_authentication2/actions/workflows/style.yml/badge.svg
[🚎cov-wf]: https://github.com/VitalConnectInc/open_id_authentication2/actions/workflows/coverage.yml
[🚎cov-wfi]: https://github.com/VitalConnectInc/open_id_authentication2/actions/workflows/coverage.yml/badge.svg
[🚎hd-wf]: https://github.com/VitalConnectInc/open_id_authentication2/actions/workflows/heads.yml
[🚎hd-wfi]: https://github.com/VitalConnectInc/open_id_authentication2/actions/workflows/heads.yml/badge.svg

</div>

Provides a thin wrapper around the excellent ruby-openid gem from JanRan.

To understand what OpenID is about and how it works, it helps to read the documentation for lib/openid/consumer.rb
from that gem.

The specification used is http://openid.net/specs/openid-authentication-2_0.html.

In the early days of Rails, this was an official Rails' plugin, written by DHH. See [Credits](#credits) for more information.

## Prerequisites

OpenID authentication uses the session, so be sure that you haven't turned that off.

Alternatively, you can use the file-based store, which just relies on on tmp/openids being present in RAILS_ROOT. But be aware that this store only works if you have a single application server. And it's not safe to use across NFS. It's recommended that you use the database store if at all possible. To use the file-based store, you'll also have to add this line to your config/environment.rb:

    OpenIdAuthentication.store = :file

This particular plugin also relies on the fact that the authentication action allows for both POST and GET operations.
If you're using RESTful authentication, you'll need to explicitly allow for this in your routes.rb.

The plugin also expects to find a root_url method that points to the home page of your site. You can accomplish this by using a root route in config/routes.rb:

    root :to => "articles#index"

This plugin relies on Rails Edge revision 6317 or newer.

## Example

This example is just to meant to demonstrate how you could use OpenID authentication. You might well want to add
salted hash logins instead of plain text passwords and other requirements on top of this. Treat it as a starting point,
not a destination.

Note that the User model referenced in the simple example below has an 'identity_url' attribute. You will want to add the same or similar field to whatever
model you are using for authentication.

Also of note is the following code block used in the example below:

    authenticate_with_open_id do |result, identity_url|
      ...
    end

In the above code block, 'identity_url' will need to match user.identity_url exactly. 'identity_url' will be a string in the form of 'http://example.com' -
If you are storing just 'example.com' with your user, the lookup will fail.

There is a handy method in this plugin called 'normalize_url' that will help with validating OpenID URLs.

    OpenIdAuthentication.normalize_url(user.identity_url)

The above will return a standardized version of the OpenID URL - the above called with 'example.com' will return 'http://example.com/'
It will also raise an InvalidOpenId exception if the URL is determined to not be valid.
Use the above code in your User model and validate OpenID URLs before saving them.

config/routes.rb

    #config/routes.rb
    root :to => "articles#index"
    resource :session

app/views/sessions/new.erb

    #app/views/sessions/new.erb
    <% form_tag(session_url) do %>
      <p>
        <label for="name">Username:</label>
        <%= text_field_tag "name" %>
      </p>

      <p>
        <label for="password">Password:</label>
        <%= password_field_tag %>
      </p>

      <p>
        <!-- ...or use: -->
      </p>

      <p>
        <label for="openid_identifier">OpenID:</label>
        <%= text_field_tag "openid_identifier" %>
      </p>

      <p>
        <%= submit_tag 'Sign in', :disable_with => "Signing in&hellip;" %>
      </p>
    <% end %>


app/controllers/sessions_controller.rb

    #app/controllers/sessions_controller.rb
    class SessionsController < ApplicationController
      def create
        if using_open_id?
          open_id_authentication
        else
          password_authentication(params[:name], params[:password])
        end
      end


      protected
      def password_authentication(name, password)
        if @current_user = @account.users.authenticate(params[:name], params[:password])
          successful_login
        else
          failed_login "Sorry, that username/password doesn't work"
        end
      end

      def open_id_authentication
        authenticate_with_open_id do |result, identity_url|
          if result.successful?
            if @current_user = @account.users.find_by_identity_url(identity_url)
              successful_login
            else
              failed_login "Sorry, no user by that identity URL exists (#{identity_url})"
            end
          else
            failed_login result.message
          end
        end
      end


      private
      def successful_login
        session[:user_id] = @current_user.id
        redirect_to(root_url)
      end

      def failed_login(message)
        flash[:error] = message
        redirect_to(new_session_url)
      end
    end

If you're fine with the result messages above and don't need individual logic on a per-failure basis,
you can collapse the case into a mere boolean:

    def open_id_authentication
      authenticate_with_open_id do |result, identity_url|
        if result.successful? && @current_user = @account.users.find_by_identity_url(identity_url)
          successful_login
        else
          failed_login(result.message || "Sorry, no user by that identity URL exists (#{identity_url})")
        end
      end
    end


## Simple Registration OpenID Extension

Some OpenID Providers support this lightweight profile exchange protocol.  See more: http://www.openidenabled.com/openid/simple-registration-extension

You can support it in your app by changing #open_id_authentication

      def open_id_authentication(identity_url)
        # Pass optional :required and :optional keys to specify what sreg fields you want.
        # Be sure to yield registration, a third argument in the
        # #authenticate_with_open_id block.
        authenticate_with_open_id(identity_url,
            :required => [ :nickname, :email ],
            :optional => :fullname) do |result, identity_url, registration|
          case result.status
          when :missing
            failed_login "Sorry, the OpenID server couldn't be found"
          when :invalid
            failed_login "Sorry, but this does not appear to be a valid OpenID"
          when :canceled
            failed_login "OpenID verification was canceled"
          when :failed
            failed_login "Sorry, the OpenID verification failed"
          when :successful
            if @current_user = @account.users.find_by_identity_url(identity_url)
              assign_registration_attributes!(registration)

              if current_user.save
                successful_login
              else
                failed_login "Your OpenID profile registration failed: " +
                  @current_user.errors.full_messages.to_sentence
              end
            else
              failed_login "Sorry, no user by that identity URL exists"
            end
          end
        end
      end

      # registration is a hash containing the valid sreg keys given above
      # use this to map them to fields of your user model
      def assign_registration_attributes!(registration)
        model_to_registration_mapping.each do |model_attribute, registration_attribute|
          unless registration[registration_attribute].blank?
            @current_user.send("#{model_attribute}=", registration[registration_attribute])
          end
        end
      end

      def model_to_registration_mapping
        { :login => 'nickname', :email => 'email', :display_name => 'fullname' }
      end

## Attribute Exchange OpenID Extension

Some OpenID providers also support the OpenID AX (attribute exchange) protocol for exchanging identity information between endpoints.  See more: http://openid.net/specs/openid-attribute-exchange-1_0.html

Accessing AX data is very similar to the Simple Registration process, described above -- just add the URI identifier for the AX field to your :optional or :required parameters.  For example:

    authenticate_with_open_id(identity_url,
      :required => [ :email, 'http://schema.openid.net/birthDate' ]) do
        |result, identity_url, registration, ax|

This would provide the sreg data for :email via registration, and the AX data for http://schema.openid.net/birthDate via ax.

## TODO

- More tests

## Credits

### 🌈 Contributors

Current maintainer(s):

- [Peter Boling](https://github.com/pboling)

Special thanks to:
- David Heinemeier Hansson - author of Rails' [original `open_id_authentication`](https://github.com/rails/open_id_authentication)
- [Joshua Peek](https://github.com/josh) maintainer of Rails' [original `open_id_authentication2`](https://github.com/rails/open_id_authentication)

And all the other contributors!

[![Contributors][🖐contributors-img]][🖐contributors]

Made with [contributors-img][🖐contrib-rocks].

[🖐contrib-rocks]: https://contrib.rocks
[🖐contributors]: https://github.com/VitalConnectInc/open_id_authentication2/graphs/contributors
[🖐contributors-img]: https://contrib.rocks/image?repo=VitalConnectInc/open_id_authentication2

## 📄 License

The gem is available as open source under the terms of
the [MIT License][📄license] [![License: MIT][📄license-img]][📄license-ref].

See [LICENSE.txt][📄license] for the official [Copyright Notice][📄copyright-notice-explainer].

[comment]: <> ( 📄 LEGAL LINKS )

[📄copyright-notice-explainer]: https://opensource.stackexchange.com/questions/5778/why-do-licenses-such-as-the-mit-license-specify-a-single-year
[📄license]: LICENSE.txt
[📄license-ref]: https://opensource.org/licenses/MIT
[📄license-img]: https://img.shields.io/badge/License-MIT-green.svg
