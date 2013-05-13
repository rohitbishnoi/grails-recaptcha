# Introduction

This plugin is designed to make using the ReCaptcha and Mailhide services within Grails easy. In order to use this plugin, you must have a ReCaptcha account, available from [http://www.google.com/recaptcha](http://www.google.com/recaptcha).

# Installation

Add the following to your `grails-app/conf/BuildConfig.groovy`

    …
    plugins {
        …
        compile ':recaptcha:0.6.5'
        …
    }
    
Upon installation, run `grails recaptcha-quickstart` to create the skeleton configuration. The quickstart has two targets: `integrated` or `standalone`, depending on where you'd like the configuration to live.

## Integrated Configuration
Integrated configuration adds the following to the end of your `Config.groovy` file:

    // Added by the Recaptcha plugin:
    recaptcha {
        // These keys are generated by the ReCaptcha service
        publicKey = ""
        privateKey = ""

        // Include the noscript tags in the generated captcha
        includeNoScript = true

        // Force language change. See this for more: http://code.google.com/p/recaptcha/issues/detail?id=133
        forceLanguageInURL = false

        // Set to false to disable the display of captcha
        enabled = true

        // Communicate using HTTPS
        useSecureAPI = true
    }

    mailhide {
        // Generated by the Mailhide service
        publicKey = ""
        privateKey = ""
    }
    
This configuration can be modified to mimic the standalone if there is a need for different behavior depending on the current environment.

## Standalone Configuration
Standlaone configuration creates a file called  `RecaptchaConfig.groovy`  in  `grails-app/conf` with the following content:

	recaptcha {
	    // These keys are generated by the ReCaptcha service
	    publicKey = ""
	    privateKey = ""

	    // Include the noscript tags in the generated captcha
	    includeNoScript = true

	    // Force language change. See this for more: http://code.google.com/p/recaptcha/issues/detail?id=133
        forceLanguageInURL = false
	}

	mailhide {
	    publicKey = ""
	    privateKey = ""
	} 

	environments {
	  development {
	    recaptcha {
	      // Set to false to disable the display of captcha
	      enabled = false

	      // Communicate using HTTPS
	      useSecureAPI = false
	    }
	  }
	  production {
	    recaptcha {
	      // Set to false to disable the display of captcha
	      enabled = true

	      // Communicate using HTTPS
	      useSecureAPI = true
	    }
	  }
	}

# Usage - ReCaptcha

The plugin is simple to use. In order to use it, there are four basic steps:

## Edit the Configuration

The configuration values are pretty self-explanatory, and match with values used by the ReCaptcha service. You must enter your public and private ReCaptcha keys, or errors will be thrown when trying to display a captcha.

## Use the Tag Library

The plugin includes four ReCaptcha tags:  `<recaptcha:ifEnabled>`, `<recaptcha:ifDisabled>`, `<recaptcha:recaptcha>`, and  `<recaptcha:ifFailed>`.

* The `<recaptcha:ifEnabled>` tag is a simple utility tag that will render the contents of the tag if the captcha is enabled in  `RecaptchaConfig.groovy`.
* The `<recaptcha:ifDisabled>` tag is a simple utility tag that will render the contents of the tag if the captcha is disabled in  `RecaptchaConfig.groovy`.
* The `<recaptcha:recaptcha>` tag is responsible for generating the correct HTML output to display the captcha. It supports four attributes: "theme", "lang", "tabindex", and "custom\_theme\_widget". These attributes map directly to the values that can be set according to the ReCaptcha API. See the [ReCaptcha Client Guide](https://developers.google.com/recaptcha/intro) for more details.
* The `<recaptcha:recaptchaAjax>` tag is responsible for creating the correct HTML output to display the captcha in an AJAX manner. The tag creates a JavaScript method called `showRecaptcha` that takes an element name as a parameter. This element will contain the generated ReCaptcha widget. This tag supports the same attributes as the `<recaptcha:recaptcha>` tag.
* The `<recaptcha:ifFailed>` tag will render its contents if the previous validation failed. Some ReCaptcha themes, like "clean", do not display error messages and require the developer to show an error message. Use this tag if you're using one of these themes.

## Verify the Captcha

In your controller, call `recaptchaService.verifyAnswer(session, request.getRemoteAddr(), params)` to verify the answer provided by the user. This method will return true or false, but will set the `error_message` property on the captcha behind the scenes so that the error message will be properly displayed when the ReCaptcha is redisplayed. Also note that `verifyAnswer` will return `true` if the plugin has been disabled in the configuration - this means you won't have to change your controller.

## Clean up After Yourself

Once the captcha has been verified, call `recaptchaService.cleanUp(session)`. This is not strictly needed, but it will clean the errors from the session.

## Examples

Here's a simple example pulled from an account creation application.

### Tag Usage

In `create.gsp`, we add the code to show the captcha:

    <recaptcha:ifEnabled>
        <recaptcha:recaptcha theme="blackglass"/>
    </recaptcha:ifEnabled>

In this example, we're using ReCaptcha's "blackglass" theme. Leaving out the "theme" attribute will default the captcha to the "red" theme.

If you are using a theme that does not supply error messages, your code might look like this:

    <recaptcha:ifEnabled>
        <recaptcha:recaptcha theme="clean"/>
        <recaptcha:ifFailed>CAPTCHA Failed</recaptcha:ifFailed>
    </recaptcha:ifEnabled>
    
### AJAX Tag Usage
    <g:form action="someAction">
      <recaptcha:ifEnabled>
          <recaptcha:recaptchaAjax theme="blackglass"/>
      </recaptcha:ifEnabled>
      <g:submitButton name="show" onclick="showRecaptcha(mydiv); return false;"/>
      <g:submitButton value="Submit" name="submit"/><br/>
      <div id="mydiv"></div>
      <recaptcha:ifFailed>CAPTCHA Failed</recaptcha:ifFailed>
    </g:form>
    
When the `show` button is clicked, the ReCaptcha widget will be created and displayed in the `mydiv` element. **The div used to display the widget must be enclosed in the `<g:form>` or the parameters will not be captured correctly.**

It is recommended to use the `<recaptcha:ifFailed>` tag in conjunction with the AJAX tag.

### Customizing the Language

If you want to change the language your captcha uses, there are two routes you can follow.

* Set `lang = "someLang"` in the `<recaptcha/>` tag. This will show the desired language if the user has their browser set to display that language.
* Set `forceLanguageInURL = true` in `ReCaptchaConfig.groovy` in addition to the above step. This will add another parameter to the generated URL, forcing the captcha to be shown in the desired language.

See [this discussion](http://code.google.com/p/recaptcha/issues/detail?id=133) for more information about changing the language.
See [ReCaptcha Customization Guide](https://developers.google.com/recaptcha/docs/customization) for available languages.

### Verify User Input

Here's an abbreviated controller class that verifies the captcha value when a new user is saved:

	import com.megatome.grails.RecaptchaService
	class UserController {
		RecaptchaService recaptchaService

		def save = {
			def user = new User(params)
			...other validation...
			def recaptchaOK = true
			if (!recaptchaService.verifyAnswer(session, request.getRemoteAddr(), params)) {
				recaptchaOK = false
			}
			if(!user.hasErrors() && recaptchaOK && user.save()) {
				recaptchaService.cleanUp(session)
				...other account creation acivities...
				render(view:'showConfirmation',model:[user:user])
			}
			else {
				render(view:'create',model:[user:user])
			}
		}
	}


### Sample Using a Custom Theme


	<g:form action="validateCustom" method="post" >
	    <div id="recaptcha_widget" style="display:none">
	        <div id="recaptcha_image" style="width:300px;height:57px;"></div>
	        <div class="recaptcha_only_if_incorrect_sol" style="color:red;">
	            Incorrect Answer
	        </div>
	        Enter the words above:
	        <input id="recaptcha_response_field" name="recaptcha_response_field" type="text" autocomplete="off"/>
	        <div>
	            <a href="javascript:Recaptcha.reload()">Get another CAPTCHA</a>
	        </div>
	        <div class="recaptcha_only_if_image">
	            <a href="javascript:Recaptcha.switch_type('audio')">Get an audio CAPTCHA</a>
	        </div>
	        <div>
	            <a href="javascript:Recaptcha.showhelp()">Help</a>
	        </div>
	   </div>
	   <recaptcha:ifEnabled>
	       <recaptcha:recaptcha theme="custom" lang="en" custom_theme_widget="recaptcha_widget"/>
	   </recaptcha:ifEnabled>
	   <br/>
	   <g:submitButton name="submit"/>
	</g:form>


### Testing

Starting with version 0.4.5, the plugin should be easier to integrate into test scenarios. You can look at the test cases in the plugin itself, or you can implement something similar to:

	private void buildAndCheckAnswer(def postText, def expectedValid, def expectedErrorMessage) {
	    def mocker = new MockFor(Post.class)
	    mocker.demand.getQueryString(4..4) { new QueryString() }
	    mocker.demand.getText { postText }
	    mocker.use {
	      def response = recaptchaService.checkAnswer("123.123.123.123", "abcdefghijklmnop", "response")

	      assertTrue response.valid == expectedValid
	      assertEquals expectedErrorMessage, response.errorMessage
	    }
	}


The `postText` parameter represents the response from the ReCaptcha server. Here are examples of simulating success and failure results:

	public void testCheckAnswerSuccess() {
	    // ReCaptcha server will return true to indicate success
	    buildAndCheckAnswer("true", true, null)
	}

	public void testCheckAnswerFailure() {
	    // ReCaptcha server will return false, followed by the error message on a new line for failure
	    buildAndCheckAnswer("false\\nError Message", false, "Error Message")
	}


# Usage - Mailhide

## Edit the Configuration

The `recaptcha-quickstart` plugin creates basic configuration. You must enter your public and private Mailhide keys, or errors will be thrown when trying to display a Mailhide link.

## Use the Tag Library

The plugin includes two Mailhide tags: `<recaptcha:mailhide>` and `<recaptcha:mailhideURL>`.

* The `<recaptcha:mailhide>` tag creates a Mailhide URL that opens in a new, pop-up window per the Mailhide specification. It supports one attribute: "emailAddress", to specify the email to be hidden. The link will be created around whatever content is in the body of the tag.
* The `<recaptcha:mailhideURL>` tag creates a "raw" URL that can be used however desired. This is useful if the pop-up behavior of the other tag is not wanted. It supports two attributes: "emailAddress" and "var". The "emailAddress" attribute specifies the email to be hidden, while the "var" attribute specifies the name of the variable that the created URL should be available under in the page. The URL variable is only available in the context of the tag body.

## Examples

### mailhide tag

    <recaptcha:mailhide emailAddress="x@example.com">Some text to wrap in a link</recaptcha:mailhide>


will create:


	<a href="http://www.google.com/recaptcha/mailhide/d?k=...publicKey...&c=..encryptedEmail..."
	     onclick="window.open('http://www.google.com/recaptcha/mailhide/d?k=...publicKey...&c=...encryptedEmail...', '', 
	     'toolbar=0,scrollbars=0,location=0,statusbar=0,menubar=0,resizable=0,width=500,height=300'); return false;" 
	     title="Reveal this e-mail address">Some text to wrap in a link</a>


### mailhideURL tag

    <recaptcha:mailhideURL emailAddress="x@example.com" var="mu">
        Created Mailhide URL: ${mu}
    </recaptcha:mailhideURL>


will create:


    Created Mailhide URL: http://www.google.com/recaptcha/mailhide/d?k=...publicKey...&c=...encryptedEmail...


# Misc.


### CHANGELOG

* 0.6.5 Don't crash when boolean configuration options are missing. ([GitHub Issue #10](https://github.com/iamthechad/grails-recaptcha/issues/10)) Establish defaults for boolean options in case they go missing. ([GitHub Issue #11](https://github.com/iamthechad/grails-recaptcha/issues/11)) Don't crash when creating an AJAX captcha. ([GitHub Issue #12](https://github.com/iamthechad/grails-recaptcha/issues/12))
* 0.6.4 Ensure that true/false settings are loaded correctly from a .properties file. ([GitHub Issue #9](https://github.com/iamthechad/grails-recaptcha/issues/9))
* 0.6.3 Ensure that AJAX tags properly use HTTPS when specified. ([GitHub Issue #7](https://github.com/iamthechad/grails-recaptcha/issues/7))
* 0.6.2 Remove spurious `println` left over. Change install behavior to not create `RecaptchaConfig.groovy` in `_Install.groovy`. Add new script `recaptcha-quickstart` to handle creation of required configuration. 
* 0.6.0 Add the ability to display the widget using AJAX. Change plugin to require Grails 2.0 at a minimum.
* 0.5.3 Add the ability to force a different language to be displayed.
* 0.5.1 & 0.5.2 Update to use the new ReCaptcha URLs.
* 0.5.0 Add Mailhide support. Add support for specifying configuration options elsewhere than `RecaptchaConfig.groovy` via the `grails.config.locations` method.
* 0.4.5 Add code to perform the ReCaptcha functionality - removed recaptcha4j library. Don't add captcha instance to session to avoid serialization issues. Hopefully make it easier to test against.
* 0.4 New version number for Grails 1.1. Same functionality as 0.3.2
* 0.3.2 Moved code into packages. Tried to make licensing easier to discern. Updated to Grails 1.0.4
* 0.3 Added support for environments and new `<recaptcha:ifFailed>` tag. Updated to Grails 1.0.3
* 0.2 initial release, developed and tested against Grails 1.0.2

### KNOWN ISSUES

* If you are testing locally on a Mac, you may need to change `recaptchaService.verifyAnswer(session, request.getRemoteAddr(), params)` to `recaptchaService.verifyAnswer(session, "127.0.0.1", params)`. This seems to be an issue with the ReCaptcha service not being able to handle the IPV6 localhost identifier.

### TODO

* Automate session cleanup

### Thanks

* The `recaptcha-quickstart` script was borrowed heavily from the [Spring Security Core plugin](http://grails.org/plugin/spring-security-core).


# Suggestions or Comments

Feel free to submit questions or comments to the Grails users mailing list.

[http://grails.org/Mailing+lists](http://grails.org/Mailing+lists)

Alternatively you can contact me directly - cjohnston at megatome dot com
