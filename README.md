# i18n
## Smart internationalization for .NET web apps
```
    PM> Install-Package I18N
```
### Introduction

The i18n library is designed to replace the use of .NET resources in favor 
of an easier, globally recognized standard for localizing ASP.NET web applications. 

### Features
- Globally recognized interface; localize like the big kids
- Localizes everything: HTML, Razor, C#, JavaScript, ...
- SEO-friendly; language selection varies the URL, and `Content-Language` is set appropriately
- Automatic; no URL/routing changes required in the app
- High performance, minimal overhead and minimal heap allocations
- Smart; knows when to hold them, fold them, walk away, or run, based on i18n best practices

### Project Configuration

The i18n library works by modifying your HTTP traffic to perform string replacement and
patching of URLs with language tags ([URL Localization](#url-localization)). The work is done by an
HttpModule called i18n.LocalizingModule which should be enabled in your web.config file as follows:

```xml
  <system.web>
    <httpModules>
      <add name="i18n.LocalizingModule" type="i18n.LocalizingModule, i18n" />
    </httpModules>
  </system.web>
  <system.webServer> <!-- IIS7 'Integrated Mode'-specific config stuff -->
    <modules>
      <add name="i18n.LocalizingModule" type="i18n.LocalizingModule, i18n" />
    </modules>
  </system.webServer>
```

Note: The ```<system.webServer>``` element is added for completeness and may not be required.

The following ```<appSettings>``` are then required to specify the type and location 
of your application's source files:

```xml
  <appSettings>
    <add key="i18n.DirectoriesToScan" value=".." /> <!-- Rel to web.config file -->
    <add key="i18n.WhiteList" value="*.cs;*.cshtml;*.sitemap" />
  </appSettings>
```

Finally, certain behaviours of i18n may be altered at runtime on application startup. The following
code shows the most common options:

```csharp
    public class MvcApplication : System.Web.HttpApplication
    {
        protected void Application_Start()
        {
            // Change from the default of 'en'.
            i18n.LocalizedApplication.Current.DefaultLanguage = "fr";

            // Change from the of temporary redirects during URL localization
            i18n.LocalizedApplication.Current.PermanentRedirects = true;

            // This line can be used to disable URL Localization.
            //i18n.LocalizedApplication.Current.EarlyUrlLocalizerService = null;

            // Change the URL localization scheme from Scheme1.
            i18n.UrlLocalizer.UrlLocalizationScheme = i18n.UrlLocalizationScheme.Scheme2;

            // Blacklist certain URLs from being 'localized'.
            i18n.UrlLocalizer.IncomingUrlFilters += delegate(Uri url) {
                if (url.LocalPath.EndsWith("sitemap.xml", StringComparison.OrdinalIgnoreCase)) {
                    return false; }
                return true;
            };
        }
    }
```

### Usage
To localize text in your application, surround your strings with [[[ and ]]] markup
characters to mark them as translatable. That's it. 
Here's an example of localizing text in a Razor view:

```html
    <div id="content">
        <h2>[[[Welcome to my web app!]]]</h2>
        <h3><span>[[[Amazing slogan here]]]</span></h3>
        <p>[[[Ad copy that would make Hiten Shah fall off his chair!]]]</p>
        <span class="button" title="[[[Click to see plans and pricing]]]">
            <a href="@Url.Action("Plans", "Home", new { area = "" })">
                <strong>[[[SEE PLANS & PRICING]]]</strong>
                <span>[[[Free unicorn with all plans!]]]</span>
            </a>
        </span>
    </div>
```

And here's an example in a controller:

```csharp
    using i18n;
    
    namespace MyApplication
    {
        public class HomeController : LocalizingController
        {
            public ActionResult Index()
            {
                ViewBag.Message = "[[[Welcome to ASP.NET MVC!]]]";

                return View();
            }
        }
    }
```

For use in data annotations:

```csharp
    public class PasswordResetViewModel
    {
        [Required(ErrorMessage="[[[Please fill in this field]]]")]
        [Email(ErrorMessage = "[[[Email not yet correct]]]")]
        [Display(Name = "[[[Email Address]]]")]
        public string Email
        {
            get;
            set;
        }
    }
```

For use in MVC URL-Helpers or other functions that require a plain string:

```html
@Html.LabelFor(m => m.Name, "[[[First Name]]]")
```

And the same can be used for Javascript:

```html
    <script type="text/javascript">
        $(function () {
            alert("[[[Hello world!]]]");
        });
    </script>
```

### Nuggets

Strings you want to be translatable are known as messages. These in turn are 'marked-up' in your
source code as 'Nuggets'. The nugget markup allows i18n to filter the HTTP response looking for the
message strings which are replaced with translated strings, where available.

A simple nugget looks like this:

```
[[[translate me]]]
```

This defines a message with the key "translate me".

Nugget markup supports formated messages as follows:

```
string.Format("[[[welcome %1, today is %0|||{0}|||{1}]]]", day, name)
```

where the %0 and %1 tokens are replaced by the strings that replace the {0} and {1} items, respectively.
(The reason for the extra level of redirection here is to facilitate the translator rearranging the order of
the tokens for different languages.)

Nugget markup also supports comments (_extracted comments_ in PO parlance) to be passed to the translator like so:

```
[[[translate me///this is an extracted comment]]]
```

#### Nugget markup customization

The character sequences for marking-up nuggets ([[[, ]]], ||| and ///) were chosen on the basis that they were unlikely to clash with
common character sequences in HTML markup while at the same time being convenient for the programmer
to enter (on most keyboards).

However, recognizing that a clash remains possible and nuggets thereby being falsely detected
in source code or the HTML response, i18n allows you to define your own sequences for the markup
which you know are not going to clash. You can configure these in web.config as shown in the following example:

```xml
  <appSettings>
    ...
    <add key="i18n.NuggetBeginToken" value="[&[" />
    <add key="i18n.NuggetEndToken" value="]&]" />
    <add key="i18n.NuggetDelimiterToken" value="||||" />
    <add key="i18n.NuggetCommentToken" value="////" />
    ...
  </appSettings>
```

### Building PO databases

To set up automatic PO database building, add the following post-build task to your project, after
adding `i18n.PostBuild.exe` as a project reference:

```
    "$(TargetDir)i18n.PostBuild.exe" "$(ProjectDir)\web.config"
```
    
Alternatively, you may choose to install the `i18n.POTGenerator.vsix` Visual Studio 2012 extension.
This installs an `i18n` button in the Solution Window for manual triggering of PO generation. Note that
it is necessary to highlight the project in question within the Solution Window before pressing the button.

The PO generator will rip through your source code (as defined by the
i18n.DirectoriesToScan and i18n.WhiteList settings in web.config), finding every nugget, 
and uses this to build a master .POT template file located at `locale/messages.pot`
relative to your web application folder. After the new template is constructed, any locales that exist 
inside the `locale` folder (or as defined by the i18n.AvailableLanguages semi-colon-delimited web.config setting)
are automatically merged with the template, so that new strings can be flagged for further translation.

From here, you can use any of the widely available PO editing tools (like [POEdit](http://www.poedit.net))
to provide locale-specific text and place them in your `locale` folder relative to the provided language, e.g. `locale/fr`. 
If you change a PO file on the fly, i18n will update accordingly; you do _not_ need to restart your application.

### URL Localization

In keeping with emerging standards for internationalized web applications, i18n provides support for
localized URLs. For example, `www.example.com/de` or `www.example.com/en-us/signin`.

Out of the box, i18n will attempt to ensure the current language for any request is shown correctly in the
address box of the user's browser, redirecting from any non-localized URL if necessary to a localized one.
This is known as [Early URL Localization](https://docs.google.com/drawings/d/1cH3_PRAFHDz7N41l8Uz7hOIRGpmgaIlJe0fYSIOSZ_Y/edit?pli=1).
See also [Principal Application Language](#principal-application-language).

While URLs from the user-agent perspective are localized, from the app's perspective they are nonlocalized.
Thus you can write your app without worrying about the language tag in the URL.

The default URL Localization scheme (Scheme1) will show the language tag in the URL always; an alternative
scheme, Scheme2, will show the language tag only if it is not the default. Alternatively, URL localization
can be disabled by setting `i18n.LocalizedApplication.Current.EarlyUrlLocalizerService = null` in `Application_Start`.

### Principal Application Language

During startup of your ASP.NET application, i18n determines the set of application 
languages for which one or more translated messages exist.

Then, on each request, one of these languages is selected as the Principal Application 
Language (PAL) for the request.

The PAL for the request is determined by the first of the following conditions that is met:

1. The path component of the URL is prefixed with a language tag that matches *exactly* one of the application languages. E.g. "example.com/fr/account/signup".

2. The path component of the URL is prefixed with a language tag that matches *loosely* one of the application languages.

3. The request contains a cookie called "i18n.langtag" with a language tag that matches (exactly or loosely) one of the application languages.

4. The request contains an Accept-Language header with a language that matches (exactly or loosely) one of the application languages.

5. The default application language is selected.

Where a *loose* match is made above, the URL is updated with the matched application language tag
and a redirect is issued. E.g. "example.com/fr-CA/account/signup" -> "example.com/fr/account/signup".
By default this is a temporary 302 redirect, but you can choose for it to be a permanent 301 one
by setting `i18n.LocalizedApplication.Current.PermanentRedirects = true` in Application_Start.

The `GetPrincipalAppLanguageForRequest` extension method to HttpContext can be called to access the
PAL of the current request. For example, it may be called in a Razor view as follows to display
the current langue to the user:

```xml
    <div>
        <p id="lang_cur" title="@Context.GetPrincipalAppLanguageForRequest()">
            @Context.GetPrincipalAppLanguageForRequest().GetNativeNameTitleCase()
        </p>
    </div>
```

### Explicit User Language Selection

You probably want to allow users to override their browser language settings by providing a language selection
feature in your application. There are two parts to implementing this feature with i18n which revolve around
the setting of a cookie called `i18n.langtag`.

Firstly, you need to provide HTML that displays the current language and allows the user to explicitly select
a language (from those ApplicationLanguages available). An example of how to do that in ASP.NET MVC and Razor follows:

```xml
@using i18n
...
<div id="language">
  <div>
    <p id="lang_cur" title="@Context.GetPrincipalAppLanguageForRequest()">@Context.GetPrincipalAppLanguageForRequest().GetNativeNameTitleCase()</p>
  </div>
  <div id="lang_menu" style="display: none;">
    <table class="table_grid">
      <tbody>
        @{
          int i;
          int maxcols = 3;
          KeyValuePair<string, i18n.LanguageTag>[] langs = LanguageHelpers.GetAppLanguages().OrderBy(x => x.Key).ToArray();
          int cellcnt = langs.Length +1;
          for (i = 0; i < cellcnt;) {
            bool lastRow = i + maxcols >= cellcnt;
            <tr class="@(Html.Raw((i % 2) == 0 ? "even":"odd")) @(Html.Raw(lastRow ? "last":""))">
              @for (int j = 0; j < maxcols && i < cellcnt; ++i, ++j) {
                string langtag;
                string title;
                string nativelangname;
                if (i == 0) {
                  langtag = "";
                  title = "[[[Browser default language setting]]]";
                  nativelangname = "[[[Auto]]]";
                }
                else {
                  i18n.LanguageTag lt = langs[i -1].Value;
                  title = langtag = lt.ToString();
                  nativelangname = lt.NativeNameTitleCase;
                }
                <td>
                  @Html.ActionLink(
                    linkText: nativelangname, 
                    actionName: "SetLanguage", 
                    controllerName: "Account", 
                    routeValues: new { langtag = langtag, returnUrl = Request.Url },
                    htmlAttributes: new { title = title } )
                </td>
              }
              @* Fill last row with empty cells if ness, so that borders are added and balanced out. *@
              @if (lastRow) {
                for (; i % maxcols != 0; ++i) {
                  <td></td>
                }
              }
            </tr>
          }
        }
      </tbody>
    </table>
  </div>
</div>
```

On selection of a language in the above code, the AccountController.SetLanguage method is called, an example of
which follows:

```csharp

    using i18n;
    ...

    //
    // GET: /Account/SetLanguage

    [AllowAnonymous]
    public ActionResult SetLanguage(string langtag, string returnUrl)
    {
        // If valid 'langtag' passed.
        i18n.LanguageTag lt = i18n.LanguageTag.GetCachedInstance(langtag);
        if (lt.IsValid()) {
            // Set persistent cookie in the client to remember the language choice.
            Response.Cookies.Add(new HttpCookie("i18n.langtag")
            {
                Value = lt.ToString(),
                HttpOnly = true,
                Expires = DateTime.UtcNow.AddYears(1)
            });
        }
        // Owise...delete any 'language' cookie in the client.
        else {
            Response.Cookies["i18n.langtag"].FlagForRemoval(); }
        // Update PAL setting so that new language is reflected in any URL patched in the 
        // response (Late URL Localization).
        HttpContext.SetPrincipalAppLanguageForRequest(lt);
        // Patch in the new langtag into any return URL.
        if (returnUrl.IsSet()) {
            returnUrl = LocalizedApplication.Current.UrlLocalizerForApp.SetLangTagInUrlPath(returnUrl, UriKind.RelativeOrAbsolute, lt == null ? null : lt.ToString()).ToString(); }
        // Redirect user agent as approp.
        return this.RedirectWithSubSite(returnUrl);
    }
```

### Language Matching

Language matching is performed when a list of one or more user-preferred languages is matched against
a list of one or more application laguages, the goal being to choose the application languages
which the user is most likely to understand. The algorithm for this is multi-facted and multi-pass and takes the Language, 
Script and Region subtags into account.

Matching is performed once per-request to determine the [Principal Application Language](#principal-application-language)
for the request, and also once per message to be translated (aka GetText call). 
The multi-pass approach ensures a thorough attempt is made at matching a user's list of preferred 
languages (from their Accept-Language HTTP header). E.g. in the context of the following request:

```
User Languages: fr-CH, fr-CA  
Application Languages: fr-CA, fr, en
```

*fr-CA* will be matched first, and if no resource exists for that language, *fr* is tried, and failing
that, the default language *en* is fallen back on.

In recognition of the potential bottleneck of the GetText call (which typically is called many times per-request),
the matching algorithm is efficient for managed code (lock-free and essentially heap-allocation free).

Note that the following Chinese languages tags are normalized: zh-CN to zh-Hans, and zh-TW to zh-Hant.
It is still safe to use zh-CN and zh-TW, but internally they will be treated as equivalent to their new forms.

##### Language Matching Update

The latest refinement to the language matching algoritm:

```csharp
        // Principle Application Language (PAL) Prioritization:
        //   User has selected an explicit language in the webapp e.g. fr-CH (i.e. PAL is set to fr-CH).
        //   Their browser is set to languages en-US, en, zh-Hans.
        //   Therefore, UserLanguages[] equals fr-CH, en-US, zh-Hans.
        //   We don't have a particular message in fr-CH, but have it in fr and fr-CA.
        //   We also have message in en-US and zh-Hans.
        //   Surely, the message from fr or fr-CA is better match than en-US or zh-Hans.
        //   However, without PAL prioritization, en-US is returned and failing that, zh-Hans.
        //   Therefore, for the 1st entry in UserLanguages (i.e. explicit user selection in app)
        //   we try all match grades first. Only if there is no match whatsoever for the PAL
        //   do we move no to the other (browser) languages, where return to prioritizing match grade
        //   i.e. loop through all the languages first at the strictest match grade before loosening 
        //   to the next match grade, and so on.
```

### A reminder about folders in a web application

Your `locale` folder is exposed to HTTP requests as-is, just like a typical log directory, so remember to block all requests
to this folder by adding a `Web.config` file. 

```xml
    <?xml version="1.0"?>
    <configuration>    
        <system.web>
            <httpHandlers>
                <add path="*" verb="*" type="System.Web.HttpNotFoundHandler"/>
            </httpHandlers>
        </system.web>
        <system.webServer>
            <handlers>
                <remove name="BlockViewHandler"/>
                <add name="BlockViewHandler" path="*" verb="*" preCondition="integratedMode" type="System.Web.HttpNotFoundHandler"/>
            </handlers>
        </system.webServer>
    </configuration>
```

### Contributing

There's lot of room for further enhancements and features to this library, and you are encouraged to fork it and
contribute back anything new. Specifically, these would be great places to add more functionality:

* Help me fix the bugs! Chances are I don't ship in your language. Fix what hurts. Please?
* Better parsing and handling of PO files for more general purposes / outside editors
* Input and ideas on a safe universal nugget syntax (see issue #69).
