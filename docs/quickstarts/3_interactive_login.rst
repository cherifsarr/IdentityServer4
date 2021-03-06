.. _refImplicitQuickstart:
Adding User Authentication with OpenID Connect
==============================================

In this quickstart we want to add support for interactive user authentication via the
OpenID Connect protocol to our IdentityServer.

Once that is in place, we will create an MVC application that will use IdentityServer for 
authentication.

Adding the UI
^^^^^^^^^^^^^
All the protocol support needed for OpenID Connect is already built into IdentityServer.
You need to provide the necessary UI parts for login, logout, consent and error.

While the look & feel as well as the exact workflows will probably always differ in every
IdentityServer implementation, we provide a sample UI that you can use as a starting point.

This UI can be found in the `Quickstart UI repo <https://github.com/IdentityServer/IdentityServer4.Quickstart.UI>`_.
You can either clone or download this repo and drop the controllers, views, models and CSS into your web application.

Alternatively you can run this command from the command line in your web application to
automate the download::

    iex ((New-Object System.Net.WebClient).DownloadString('https://raw.githubusercontent.com/IdentityServer/IdentityServer4.Quickstart.UI/dev/get.ps1'))

Spend some time inspecting the controllers and models, the better you understand them, 
the easier it will be to make future modifications.

Creating an MVC client
^^^^^^^^^^^^^^^^^^^^^^
Next you will add an MVC application to your solution.
Use the ASP.NET Core "Web Application" template for that.
Configure the application to use port 5002 (see the overview part for instructions on how to do that).

To add support for OpenID Connect authentication to the MVC application, add the following
packages to project.json::

    "Microsoft.AspNetCore.Authentication.Cookies": "1.0.0",
    "Microsoft.AspNetCore.Authentication.OpenIdConnect": "1.0.0"

Next add both middlewares to your pipeline - the cookies one is simple::

    app.UseCookieAuthentication(new CookieAuthenticationOptions
    {
        AuthenticationScheme = "Cookies"
    });

The OpenID Connect middleware needs slightly more configuration.
You point it to your IdentityServer, specify a client ID and tell it which middleware will do
the local signin (namely the cookies middleware).  As well, we've turned off the JWT claim type mapping to allow well-known claims (e.g. 'sub' and 'idp') to flow through unmolested.  This "clearing" of the claim type mappings must be done before the call to `UseOpenIdConnectAuthentication()`::

    JwtSecurityTokenHandler.DefaultInboundClaimTypeMap.Clear();

    app.UseOpenIdConnectAuthentication(new OpenIdConnectOptions
    {
        AuthenticationScheme = "oidc",
        SignInScheme = "Cookies",

        Authority = "http://localhost:5000",
        RequireHttpsMetadata = false,

        ClientId = "mvc",
        SaveTokens = true
    });

Both middlewares should be added before the MVC in the pipeline.

The last step is to trigger the authentication handshake. For that go to the home controller and
add the ``[Authorize]`` on one of the actions.
Also modify the view of that action to display the claims of the user, e.g.::

    <dl>
        @foreach (var claim in User.Claims)
        {
            <dt>@claim.Type</dt>
            <dd>@claim.Value</dd>
        }
    </dl>

If you now navigate to that controller using the browser, a redirect attempt will be made
to IdentityServer - this will result in an error because the MVC client is not registered yet.

Adding support for OpenID Connect Scopes
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Similar to OAuth 2.0, OpenID Connect also uses the scopes concept.
Again, scopes represent something you want to protect and that clients want to access.
In contrast to OAuth, scopes in OIDC don't represent APIs, but identity data like user id, 
name or email address.

Add support for the standard ``openid`` (subject id) and ``profile`` (first name, last name etc..) scopes
by adding these scopes to your scopes configuration::

    public static IEnumerable<Scope> GetScopes()
    {
        return new List<Scope>
        {
            StandardScopes.OpenId,
            StandardScopes.Profile,

            new Scope
            {
                Name = "api1",
                Description = "My API"
            }
        };
    }

.. note:: All standard scopes and their corresponding claims can be found in the OpenID Connect `specification <https://openid.net/specs/openid-connect-core-1_0.html#ScopeClaims>`_

Adding a client for OpenID Connect implicit flow
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The last step is to add a new client to IdentityServer.

OpenID Connect-based clients are very similar to the OAuth 2.0 clients we added so far.
But since the flows in OIDC are always interactive, we need to add some redirect URLs to our configuration.

Add the following to your clients configuration::

    public static IEnumerable<Client> GetClients()
    {
        return new List<Client>
        {
            // other clients omitted...

            // OpenID Connect implicit flow client (MVC)
            new Client
            {
                ClientId = "mvc",
                ClientName = "MVC Client",
                AllowedGrantTypes = GrantTypes.Implicit,
                
                // where to redirect to after login
                RedirectUris = { "http://localhost:5002/signin-oidc" },

                // where to redirect to after logout
                PostLogoutRedirectUris = { "http://localhost:5002" },

                AllowedScopes = new List<string>
                {
                    StandardScopes.OpenId.Name,
                    StandardScopes.Profile.Name
                }
            }
        };
    }

Testing the client
^^^^^^^^^^^^^^^^^^
Now finally everything should be in place for the new MVC client.

Trigger the authentication handshake by navigating to the protected controller action.
You should see a redirect to the login page at IdentityServer.

.. image:: images/3_login.png

After successful login, the user is presented with the consent screen.
Here the user can decide if he wants to release his identity information to the client application.

.. note:: Consent can be turned off on a per client basis using the ``RequireConsent`` property on the client object.

.. image:: images/3_consent.png

..and finally the browser redirects back to the client application, which shows the claims
of the user.

.. image:: images/3_claims.png

.. note:: During development you might sometimes see an exception stating that the token could not be validated. This is due to the fact that the signing key material is created on the fly and kept in-memory only. This exception happens when the client and IdentityServer get out of sync. Simply repeat the operation at the client, the next time the metadata has caught up, and everything should work normal again.

Adding sign-out
^^^^^^^^^^^^^^^
The very last step is to add sign-out to the MVC client.

With an authentication service like IdentityServer, it is not enough to clear the local application cookies.
In addition you also need to make a roundtrip to IdentityServer to clear the central single sign-on session.

The exact protocol steps are implemented inside the OpenID Connect middleware, 
simply add the following code to some controller to trigger the sign-out::

    public async Task Logout()
    {
        await HttpContext.Authentication.SignOutAsync("Cookies");
        await HttpContext.Authentication.SignOutAsync("oidc");
    }

This will clear the local cookie and then redirect to IdentityServer.
IdentityServer will clear its cookies and then give the user a link to return back to the MVC application.

Further experiments
^^^^^^^^^^^^^^^^^^^
As mentioned above, the OpenID Connect middleware asks for the *profile* scope by default.
This scope also includes claims like *name* or *website*.

Let's add these claims to the user, so IdentityServer can put them into the identity token::

    public static List<InMemoryUser> GetUsers()
    {
        return new List<InMemoryUser>
        {
            new InMemoryUser
            {
                Subject = "1",
                Username = "alice",
                Password = "password",

                Claims = new []
                {
                    new Claim("name", "Alice"),
                    new Claim("website", "https://alice.com")
                }
            },
            new InMemoryUser
            {
                Subject = "2",
                Username = "bob",
                Password = "password",

                Claims = new []
                {
                    new Claim("name", "Bob"),
                    new Claim("website", "https://bob.com")
                }
            }
        };
    }

Next time you authenticate, your claims page will now show the additional claims.

Feel free to add more claims - and also more scopes. The ``Scope`` property on the OpenID Connect 
middleware is where you configure which scopes will be sent to IdentityServer during authentication.

It is also noteworthy, that the retrieval of claims for tokens is an extensibility point - ``IProfileService``.
Since we are using the in-memory user store, the ``InMemoryUserProfileService`` is used by default.
You can inspect the source code `here <https://github.com/IdentityServer/IdentityServer4/blob/dev/src/IdentityServer4/Services/InMemory/InMemoryUserProfileService.cs>`_
to see how it works.
