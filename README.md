This fork exists to fix a metadata generation bug in saml_idp
https://github.com/saml-idp/saml_idp/issues/71#issuecomment-1049569197
Forked repo, applied commit c087bbb and pushed up to our NterOne fork.
Do not delete, this is being used by EDGE Focus Web.

Dan Mlodecki
16 Nov 2022
daniel@nterone.com



# Ruby SAML Identity Provider (IdP)

Forked from https://github.com/lawrencepit/ruby-saml-idp

[![Build Status](https://travis-ci.org/saml-idp/saml_idp.svg)](https://travis-ci.org/saml-idp/saml_idp)
[![Gem Version](https://badge.fury.io/rb/saml_idp.svg)](http://badge.fury.io/rb/saml_idp)

The ruby SAML Identity Provider library is for implementing the server side of SAML authentication. It allows
your application to act as an IdP (Identity Provider) using the
[SAML v2.0](http://en.wikipedia.org/wiki/Security_Assertion_Markup_Language)
protocol. It provides a means for managing authentication requests and confirmation responses for SPs (Service Providers).

This was originally setup by @lawrencepit to test SAML Clients. I took it closer to a real
SAML IDP implementation.

# Installation and Usage

Add this to your Gemfile:

    gem 'saml_idp'

## Not using rails?

Include `SamlIdp::Controller` and see the examples that use rails. It should be straightforward for you.

Basically you call `decode_request(params[:SAMLRequest])` on an incoming request and then use the value
`saml_acs_url` to determine the source for which you need to authenticate a user. How you authenticate
a user is entirely up to you.

Once a user has successfully authenticated on your system send the Service Provider a SAMLReponse by
posting to `saml_acs_url` the parameter `SAMLResponse` with the return value from a call to
`encode_response(user_email)`.

## Using rails?

Add to your `routes.rb` file, for example:

```ruby
get '/saml/auth' => 'saml_idp#new'
get '/saml/metadata' => 'saml_idp#show'
post '/saml/auth' => 'saml_idp#create'
match '/saml/logout' => 'saml_idp#logout', via: [:get, :post, :delete]
```

Create a controller that looks like this, customize to your own situation:

```ruby
class SamlIdpController < SamlIdp::IdpController
  def idp_authenticate(email, password) # not using params intentionally
    user = User.by_email(email).first
    user && user.valid_password?(password) ? user : nil
  end
  private :idp_authenticate

  def idp_make_saml_response(found_user) # not using params intentionally
    # NOTE encryption is optional
    encode_response found_user, encryption: {
      cert: saml_request.service_provider.cert,
      block_encryption: 'aes256-cbc',
      key_transport: 'rsa-oaep-mgf1p'
    }
  end
  private :idp_make_saml_response

  def idp_logout
    user = User.by_email(saml_request.name_id)
    user.logout
  end
  private :idp_logout
end
```

## Configuration

#### Signed assertions and Signed Response

By default SAML Assertion will be signed with an algorithm which defined to `config.algorithm`. Because SAML assertions contain secure information used for authentication such as NameID.

Signing SAML Response is optional, but some security perspective SP services might require Response message itself must be signed.
For that, you can enable it with `config.signed_message` option. [More about SAML spec](https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf#page=68)

#### Signing algorithm

Following algorithms you can set in your response signing algorithm
:sha1 - RSA-SHA1 default value but not recommended to production environment
Highly recommended to use one of following algorithm, suit with your computing power.
:sha256 - RSA-SHA256
:sha384 - RSA-SHA384
:sha512 - RSA-SHA512

Be sure to load a file like this during your app initialization:

```ruby
SamlIdp.configure do |config|
  base = "http://example.com"

  config.x509_certificate = <<-CERT
-----BEGIN CERTIFICATE-----
CERTIFICATE DATA
-----END CERTIFICATE-----
CERT

  config.secret_key = <<-CERT
-----BEGIN RSA PRIVATE KEY-----
KEY DATA
-----END RSA PRIVATE KEY-----
CERT

  # config.password = "secret_key_password"
  # config.algorithm = :sha256                                    # Default: sha1 only for development.
  # config.organization_name = "Your Organization"
  # config.organization_url = "http://example.com"
  # config.base_saml_location = "#{base}/saml"
  # config.reference_id_generator                                 # Default: -> { SecureRandom.uuid }
  # config.single_logout_service_post_location = "#{base}/saml/logout"
  # config.single_logout_service_redirect_location = "#{base}/saml/logout"
  # config.attribute_service_location = "#{base}/saml/attributes"
  # config.single_service_post_location = "#{base}/saml/auth"
  # config.session_expiry = 86400                                 # Default: 0 which means never
  # config.signed_message = true                                  # Default: false which means unsigned SAML Response

  # Principal (e.g. User) is passed in when you `encode_response`
  #
  # config.name_id.formats =
  #   {                         # All 2.0
  #     email_address: -> (principal) { principal.email_address },
  #     transient: -> (principal) { principal.id },
  #     persistent: -> (p) { p.id },
  #   }
  #   OR
  #
  #   {
  #     "1.1" => {
  #       email_address: -> (principal) { principal.email_address },
  #     },
  #     "2.0" => {
  #       transient: -> (principal) { principal.email_address },
  #       persistent: -> (p) { p.id },
  #     },
  #   }

  # If Principal responds to a method called `asserted_attributes`
  # the return value of that method will be used in lieu of the
  # attributes defined here in the global space. This allows for
  # per-user attribute definitions.
  #
  ## EXAMPLE **
  # class User
  #   def asserted_attributes
  #     {
  #       phone: { getter: :phone },
  #       email: {
  #         getter: :email,
  #         name_format: Saml::XML::Namespaces::Formats::NameId::EMAIL_ADDRESS,
  #         name_id_format: Saml::XML::Namespaces::Formats::NameId::EMAIL_ADDRESS
  #       }
  #     }
  #   end
  # end
  #
  # If you have a method called `asserted_attributes` in your Principal class,
  # there is no need to define it here in the config.

  # config.attributes # =>
  #   {
  #     <friendly_name> => {                                                  # required (ex "eduPersonAffiliation")
  #       "name" => <attrname>                                                # required (ex "urn:oid:1.3.6.1.4.1.5923.1.1.1.1")
  #       "name_format" => "urn:oasis:names:tc:SAML:2.0:attrname-format:uri", # not required
  #       "getter" => ->(principal) {                                         # not required
  #         principal.get_eduPersonAffiliation                                # If no "getter" defined, will try
  #       }                                                                   # `principal.eduPersonAffiliation`, or no values will
  #    }                                                                      # be output
  #
  ## EXAMPLE ##
  # config.attributes = {
  #   GivenName: {
  #     getter: :first_name,
  #   },
  #   SurName: {
  #     getter: :last_name,
  #   },
  # }
  ## EXAMPLE ##

  # config.technical_contact.company = "Example"
  # config.technical_contact.given_name = "Jonny"
  # config.technical_contact.sur_name = "Support"
  # config.technical_contact.telephone = "55555555555"
  # config.technical_contact.email_address = "example@example.com"

  service_providers = {
    "some-issuer-url.com/saml" => {
      fingerprint: "9E:65:2E:03:06:8D:80:F2:86:C7:6C:77:A1:D9:14:97:0A:4D:F4:4D",
      metadata_url: "http://some-issuer-url.com/saml/metadata",

      # We now validate AssertionConsumerServiceURL will match the MetadataURL set above.
      # *If* it's not going to match your Metadata URL's Host, then set this so we can validate the host using this list
      response_hosts: ["foo.some-issuer-url.com"]
    },
  }

  # `identifier` is the entity_id or issuer of the Service Provider,
  # settings is an IncomingMetadata object which has a to_h method that needs to be persisted
  config.service_provider.metadata_persister = ->(identifier, settings) {
    fname = identifier.to_s.gsub(/\/|:/,"_")
    FileUtils.mkdir_p(Rails.root.join('cache', 'saml', 'metadata').to_s)
    File.open Rails.root.join("cache/saml/metadata/#{fname}"), "r+b" do |f|
      Marshal.dump settings.to_h, f
    end
  }

  # Find ServiceProvider metadata_url and fingerprint based on our settings
  config.service_provider.finder = ->(issuer_or_entity_id) do
    service_providers[issuer_or_entity_id]
  end
end
```

# Keys and Secrets

To generate the SAML Response it uses a default X.509 certificate and secret key... which isn't so secret.
You can find them in `SamlIdp::Default`. The X.509 certificate is valid until year 2032.
Obviously you shouldn't use these if you intend to use this in production environments. In that case,
within the controller set the properties `x509_certificate` and `secret_key` using a `prepend_before_action`
callback within the current request context or set them globally via the `SamlIdp.config.x509_certificate`
and `SamlIdp.config.secret_key` properties.

The fingerprint to use, if you use the default X.509 certificate of this gem, is:

```
9E:65:2E:03:06:8D:80:F2:86:C7:6C:77:A1:D9:14:97:0A:4D:F4:4D
```

# Service Providers

To act as a Service Provider which generates SAML Requests and can react to SAML Responses use the
excellent [ruby-saml](https://github.com/onelogin/ruby-saml) gem.

# Author

Jon Phenow, jon@jphenow.com, jphenow.com, @jphenow

Lawrence Pit, lawrence.pit@gmail.com, lawrencepit.com, @lawrencepit

# Copyright

Copyright (c) 2012 Sport Ngin.
Portions Copyright (c) 2010 OneLogin, LLC
Portions Copyright (c) 2012 Lawrence Pit (http://lawrencepit.com)

See LICENSE for details.
