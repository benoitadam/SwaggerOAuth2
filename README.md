# SwaggerOAuth2
Add support for "oauth2" password token in swagger

# Swagger Project
https://github.com/domaindrivendev/Swashbuckle

# Add OAuth2

In "YourProjectName/App_Start/SwaggerConfig.cs" add
```
GlobalConfiguration.Configuration
    .EnableSwagger("docs/api/{apiVersion}", c =>
    {
        ...

        c.OAuth2("oauth2")
            .Description("OAuth2 Implicit Grant")
            .Flow("implicit")
            .AuthorizationUrl("/Token");
        
        c.OperationFilter<AssignOAuth2SecurityRequirements>();
    })
    .EnableSwaggerUi("docs/ui/{*assetPath}", c =>
    {
        ...

        c.InjectJavaScript(typeof(SwaggerConfig).Assembly,
            "YourProjectName.SwaggerExtensions.oauth2_token.js");
    });
```

Create "YourProjectName/SwaggerExtensions/AssignOAuth2SecurityRequirements.cs" :
```
public class AssignOAuth2SecurityRequirements : IOperationFilter
{
    public void Apply(Operation operation, SchemaRegistry schemaRegistry, ApiDescription apiDescription)
    {
        // Correspond each "Authorize" role to an oauth2 scope
        var scopes = apiDescription.ActionDescriptor.GetFilterPipeline()
            .Select(filterInfo => filterInfo.Instance)
            .OfType<AuthorizeAttribute>()
            .SelectMany(attr => attr.Roles.Split(','))
            .Distinct();

        if (scopes.Any())
        {
            if (operation.security == null)
                operation.security = new List<IDictionary<string, IEnumerable<string>>>();

            var oAuthRequirements = new Dictionary<string, IEnumerable<string>>
            {
                { "oauth2", scopes }
            };

            operation.security.Add(oAuthRequirements);
        }
    }
}
```

Create the "Embedded Resource" : "YourProjectName/SwaggerExtensions/oauth2_token.js" :
```
$(function () {

    // Add login formulaire
    var basicAuthUI = [
        '<div class="input">',
            '<input placeholder="username" id="input_username" name="username" type="text" size="10">',
        '</div>',
        '<div class="input">',
            '<input placeholder="password" id="input_password" name="password" type="password" size="10">',
        '</div>'
    ].join("\n");
    $(basicAuthUI).insertBefore('#api_selector div.input:last-child');
    $("#input_apiKey").hide();

    // On login formulaire change
    $("#input_username, #input_password").change(function () {

        // Get login information
        var username = ($('#input_username').val() || "").trim();
        var password = ($('#input_password').val() || "").trim();

        if (username == "" || password == "")
            return;

        // Get token
        $.ajax({
            type: "POST",
            url: swaggerUi.api.scheme + "://" + swaggerUi.api.host + "/token",
            data: {
                grant_type: "password",
                username: username,
                password: password
            },
            success: function (data, textStatus, jqXHR) {
                $("#input_password")
                    .css('color', 'green')
                    .css('border', '1px solid green');

                // Add token to header
                var authorization = data.token_type + " " + data.access_token;
                console.log("Authorization", authorization);

                swaggerUi.api.clientAuthorizations.add("oauth2",
                    new SwaggerClient.ApiKeyAuthorization("Authorization", authorization, "header"));
            },
            error: function (jqXHR, textStatus, errorThrown) {
                console.error("token error", jqXHR, textStatus, errorThrown);

                $("#input_password")
                    .css('color', 'red')
                    .css('border', '1px solid red');
            },
            dataType: "json"
        });
    });
});
```
