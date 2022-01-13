---
presentation:
  theme: beige.css
  width: 1024
  height: 900
  controls: true
  enableSpeakerNotes: true
---
<!-- slide  -->
## 1. Register Your API and Web App On AAD 

<!-- slide vertical=true -->
Follow the [official manual](https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-configure-app-expose-web-apis) to expose your API.

Key points:
- Copy the `Application ID URI`.

<!-- slide vertical=true -->
Follow the [official manual](https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-configure-app-access-web-apis) to register the web app.

Key points:
- Enable `Implicit Flow` for the web app
- Set the `Redirect Url` to your web app home page
- Add API to the permission list

<!-- slide -->
## 2. Configure OAuth2 Token Validator on Backend Side

<!-- slide vertical=true  -->
Assuming you are using Spring Boot and Spring Security
Add `spring-boot-starter-parent` in pom.xml 
```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>2.1.6.RELEASE</version>
  <relativePath /> <!-- lookup parent from repository -->
</parent>
```

<!-- slide vertical=true class="three-quarterz-font-size" -->
Add dependencies
```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
  </dependency>
  <!-- Spring security -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.security.oauth</groupId>
    <artifactId>spring-security-oauth2</artifactId>
    <version>2.2.4.RELEASE</version>
  </dependency>
  <dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-oauth2-jose</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-oauth2-resource-server</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.security.oauth.boot</groupId>
    <artifactId>spring-security-oauth2-autoconfigure</artifactId>
    <version>2.1.5.RELEASE</version>
  </dependency>
</dependencies>
```
<!-- slide vertical=true class="three-quarterz-font-size"  -->
Spring Security Configuration
```java
@Configuration
@EnableWebSecurity
@EnableResourceServer
public class SecurityConfig extends WebSecurityConfigurerAdapter {
  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http
      // enable JWT token validation
	    .oauth2ResourceServer()
	    	.jwt()
        .and()
		    	.authenticationEntryPoint(aadEntryPoint);
  }
}

// the customized entry point
@Component
public class AADLoginEntryPoint implements AuthenticationEntryPoint {
	@Override
	public void commence(HttpServletRequest request, HttpServletResponse response,
      AuthenticationException authException) throws IOException, ServletException {
      /*
      * On API side, we are not responsible to handle user authentication. 
      * So we simply response 401 error here for requests with invalid token.
      */
      response.sendError(HttpStatus.UNAUTHORIZED.value(), "Unauthorized");
	}
}
```

<!-- slide vertical=true class="three-quarterz-font-size"  -->
Configure JWT token validator
```java
@Bean
public JwtDecoder jwtDecoder() {
  NimbusJwtDecoder decoder = (NimbusJwtDecoder)JwtDecoders.fromOidcIssuerLocation("https://sts.windows.net/" + this.tenantId + "/");
  // verify expiry time and audience in the token
  DelegatingOAuth2TokenValidator<Jwt> tokenValidators = new DelegatingOAuth2TokenValidator<>(
      // the maximum time skew allowed if token is found expired
      new JwtTimestampValidator(Duration.ofSeconds(60)),
      new AudienceValidator());
  decoder.setJwtValidator(tokenValidators);
  return decoder;
}

class AudienceValidator implements OAuth2TokenValidator<Jwt> {
  OAuth2Error error = new OAuth2Error("invalid_token", "Invalid token.", null);
  @Override
  public OAuth2TokenValidatorResult validate(Jwt jwt) {
    // resourceId is the Application ID URI of registered api in AAD previously
    if (null != jwt.getAudience() && jwt.getAudience().stream().anyMatch(a -> a != null && a.equalsIgnoreCase(resourceId))) {
        return OAuth2TokenValidatorResult.success();
    } else {
        return OAuth2TokenValidatorResult.failure(error);
    }
  }
}
```

<!-- slide -->
## 3. Setup OAuth2 Client In Web App

<!-- slide vertical=true  -->
Add _MSAL_ dependency in `package.json`
```json
"dependencies": {
  "axios": "^0.19.2",
  "msal": "^1.2.2",
  ...
}
```

<!-- slide vertical=true  -->
Configure MSAL client

```js
import * as Msal from 'msal';

const msalConfig = {
  auth: {
    clientId: '[App id of web app registered in AAD]',
    redirectUri: '[redirect url of the web app registered in AAD]'
  },
  cache: {
    cacheLocation: 'sessionStorage'
  }
};
```

<!-- slide vertical=true class="three-quarterz-font-size"  -->
Sign in user
```js
signIn() {
  if (!this.msalInstance) {
    this.initialize();
  }
  var loginRequest = {
  };
  return new Promise((resolve, reject) => {
    if (!this.msalInstance.getAccount()) {
      this.msalInstance
        .loginRedirect(loginRequest)
        .then(resp => {
          resolve(resp);
        })
        .catch(err => {
          reject(err);
        });
    }
    resolve(this.msalInstance.getAccount());
  });
},
```
Call the signIn function while initializing the page
```js
auth.signIn().then(function(token) { // this is the id token
  var user = {
    userId: token.userName,
    userName: token.name
  }
  // do something after user sign in, e.g. render your page
})
```

<!-- slide vertical=true class="three-quarterz-font-size"  -->
Before call api, you need acquire access token
```js
acquireToken() {
  if (!this.msalInstance) {
    this.initialize();
  }
  return new Promise((resolve, reject) => {
    if (this.msalInstance.getAccount()) {
      var tokenRequest = {
        // resourceId is the App ID URI of the registered api in AAD
        scopes: [resourceId + '/.default']
      };
      this.msalInstance
        .acquireTokenSilent(tokenRequest)
        .then(r => {
          resolve(r.accessToken);
        })
        .catch(err => {
          // could also check if err instance of InteractionRequiredAuthError if you can import the class.
          if (err.name === 'InteractionRequiredAuthError') {
            return this.msalInstance
              .acquireTokenRedirect(tokenRequest)
              .then(r => {
                resolve(r.accessToken);
              })
              .catch(err => {
                reject(err);
              });
          }
        });
    } else {
      this.msalInstance.loginRedirect();
    }
  });
},
```

<!-- slide vertical=true class="three-quarterz-font-size"  -->
then call the api with the access token
```js
import axios from 'axios'
import auth from '../auth'

// request interceptors
axios.interceptors.request.use(
  async function (config) {
    config.headers = config.headers || {};
    var token = await auth.acquireToken();
    config.headers.Authorization = "Bearer " + token;
    return config;
  },
  function(err) {
    // Do something with request error
    return Promise.reject(err);
  }
);

// api request example
axios.get('http://localhost:8080/api/v1/users')
  .then()
  .catch()
```

<!-- slide vertical=true class="half-font-size"  -->
Complete auth.js for your reference
```js
import * as Msal from 'msal';
import config from './config';

const msalConfig = {
  auth: {
    clientId: config.oauth.clientId,
    redirectUri: config.oauth.redirectUri
  },
  cache: {
    cacheLocation: 'sessionStorage'
  }
};

export default {
  msalInstance: null,
  initialize() {
    // msal instance
    this.msalInstance = new Msal.UserAgentApplication(msalConfig);
    this.msalInstance.handleRedirectCallback((err) => {
      if (err) {
        console.log(err)
      }
    });
  },
  /**
   * @return {Promise.<String>} A promise that resolves to an access token for resource access
   */
  acquireToken() {
    if (!this.msalInstance) {
      this.initialize();
    }
    return new Promise((resolve, reject) => {
      if (this.msalInstance.getAccount()) {
        var tokenRequest = {
          scopes: [config.api.resourceId + '/.default']
        };
        this.msalInstance
          .acquireTokenSilent(tokenRequest)
          .then(r => {
            resolve(r.accessToken);
          })
          .catch(err => {
            // could also check if err instance of InteractionRequiredAuthError if you can import the class.
            if (err.name === 'InteractionRequiredAuthError') {
              return this.msalInstance
                .acquireTokenRedirect(tokenRequest)
                .then(r => {
                  resolve(r.accessToken);
                })
                .catch(err => {
                  reject(err);
                });
            }
          });
      } else {
        this.msalInstance.loginRedirect();
      }
    });
  },
  // to be continue...

```

<!-- slide vertical=true -->
Continued
```js
isAuthenticated() {
    return this.msalInstance && this.msalInstance.getAccount()
  },
  /**
   * Sign in user. 
   */
  signIn() {
    if (!this.msalInstance) {
      this.initialize();
    }
    var loginRequest = {
    };
    return new Promise((resolve, reject) => {
      if (!this.msalInstance.getAccount()) {
        this.msalInstance
          .loginRedirect(loginRequest)
          .then(resp => {
            resolve(resp);
          })
          .catch(err => {
            reject(err);
          });
      }
      resolve(this.msalInstance.getAccount());
    });
  },
  signOut() {
    this.msalInstance.logout();
  }
};
```

<!-- slide -->
# -End-
