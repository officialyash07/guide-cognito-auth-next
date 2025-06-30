# Authentication using AWS Cognito in Next.js

### Prequisites

1. AWS Account.
2. Cognito User Pool.

---

### Packages needed

**1.** **@aws-sdk/client-cogito-identity-provider**

```
npm install @aws-sdk/client-cognito-identity-provider
```

The `@aws-sdk/client-cognito-identity-provider` is the AWS SDK v3 package for interacting with Amazon Cognito's Identity Provider service.

**2.** **axios**

```
npm install axios
```

`Axios` is a popular promise-based HTTP client for JavaScript that works in both browsers and Node.js.

**3.** **cookie**

```
npm install cookie
```

The `cookie` package lets you parse (read) and serialize (set) cookies in Node.js or Next.js server code.

**4.** **dotenv**

```
npm install dotenv
```

`dotenv` is a Node.js library that loads environment variables from a .env file into process.env.

**5.** **react-hook-form** (optional)

```
npm install react-hook-form
```

`react-hook-form` is a library for performant, flexible and extensible forms with easy-to-use validation.

---

## Create Cognito User Pool & App Client

Create a Cognito user pool using AWS Management Console.

![user-pool-image](user-pool.png)

Create App Client for this user pool.

![app-client-image](app-client.png)

For signup, the default is email and password, if you want to add some other attribute for eg. `Full Name`, then you will have to create a **custom attribute**. The syntax for this is `custom:fullName`.

![custom-attr-image](custom-attribute.png)

There are many types of Authentication Flows in App Client settings, choose `USERNAME` & `PASSWORD` which is named as `ALLOW_USER_PASSWORD_AUTH`.

![auth-flow-image](auth-flow.png)

**Copy the values of `region`, `user-pool-id` & `app-client-id` in `env` file.**

```
AWS_REGION=ap-south-1
USER_POOL_ID=ap-south-1_xxxxxxxx
WEB_CLIENT_ID=xxxxxxxxxxxxxxxxxxxxxxxx
```

---

## Folder Structure

```
my-project/
    ├── app/
    │     ├── api/
    │     │    ├── signup/
    │     │    │    └── route.js
    │     │    ├── confirm-signup/
    │     │    │    └── route.js
    │     │    ├── resend-code/
    │     │    │    └── route.js
    │     │    ├── login/
    │     │    │    └── route.js
    │     │    ├── logout/
    │     │    │    └── route.js
    │     │    ├── auth-status/
    │     │    │    └── route.js
    │     ├── auth/
    │     │    └── page.js (auth page)
    │     ├── dashboard/
    │     │    └── page.js (dashboard page)
    │     ├── global.css
    │     ├── layout.js
    │     └── page.js (home page)
    ├── components/
    │     └── auth/
    │          ├── signup.js
    │          ├── login.js
    │          └── confirm-signup.js
    ├── context/
    │     └── auth-context.js
    ├── utils/
    │     └── cognito-config.js
    ├── .env
```

---

## Cognito Client Configuration

Now you have to configure cognito client so that you can call Cognito API using AWS Cognito SDK.

**`/utils/cognito-config.js`**

```jsx
import { CognitoIdentityProviderClient } from "@aws-sdk/client-cognito-identity-provider";

const cognitoClient = new CognitoIdentityProviderClient({
    region: process.env.AWS_REGION || "us-east-1",
});

export default cognitoClient;
```

---

## APIs Configuration

**1. Signup API**

**`/app/api/signup/route.js`**

```jsx
import { SignUpCommand } from "@aws-sdk/client-cognito-identity-provider";

import cognitoClient from "@/utils/cognito-config";

export const POST = async (req) => {
    // Grab inputs from frontend.

    const { fullName, email, password } = await req.json();

    // Input validation.

    const trimmedName = fullName.trim();
    const trimmedEmail = email.trim();
    const trimmedPassword = password.trim();

    if (!trimmedName || !trimmedEmail || !trimmedPassword) {
        return Response.json(
            { error: true, message: "All fields are required." },
            { status: 400 }
        );
    }

    const emailRegex = new RegExp(
        "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
    );
    if (!emailRegex.test(trimmedEmail)) {
        return Response.json(
            { error: true, message: "Invalid email format." },
            { status: 400 }
        );
    }

    if (trimmedPassword.length < 6) {
        return Response.json(
            { error: true, message: "Password must be at least 6 characters." },
            { status: 400 }
        );
    }

    // Check whether cognito client is configured or not.

    const ClientId = process.env.WEB_CLIENT_ID;
    if (!ClientId) {
        return Response.json(
            { error: true, message: "Client ID is not configured." },
            { status: 500 }
        );
    }

    // Sign up the user in Cognito.

    try {
        const signupParams = {
            ClientId: ClientId,
            Username: trimmedEmail,
            Password: trimmedPassword,
            UserAttributes: [
                { Name: "email", Value: trimmedEmail },
                { Name: "custom:fullName", Value: trimmedName },
            ],
        };

        const command = new SignUpCommand(signupParams);
        const response = await cognitoClient.send(command);
        console.log(response);

        return Response.json(
            {
                error: false,
                message:
                    "User signed up successfully. Please check your email to verify your account.",
            },
            { status: 200 }
        );
    } catch (error) {
        if (error.name === "UsernameExistsException") {
            return Response.json(
                { error: true, message: "Email already exists." },
                { status: 400 }
            );
        }

        console.error("Sign up error:", error);
        return Response.json(
            {
                error: true,
                message:
                    error.message ||
                    "Failed to sign up user. Please try again.",
            },
            { status: 400 }
        );
    }
};
```

**2. Confirm Signup API**

**`/app/api/confirm-signup/route.js`**

```jsx
import { ConfirmSignUpCommand } from "@aws-sdk/client-cognito-identity-provider";

import cognitoClient from "@/utils/cognito-config";

export const POST = async (req) => {
    // Grab input from frontend.

    const { email, code } = await req.json();

    //Input Validation

    const trimmedEmail = email.trim();
    const trimmedCode = code.trim();

    if (!trimmedEmail || !trimmedCode) {
        return Response.json(**``**

            {
                error: true,
                message: "Email and verification code are required.",
            },
            { status: 400 }
        );
    }

    // Check whether cognito client is configured or not.

    const ClientId = process.env.WEB_CLIENT_ID;
    if (!ClientId) {
        return Response.json(
            { error: true, message: "Client ID is not configured." },
            { status: 500 }
        );
    }

    // Verify the user using OTP.

    try {
        const confirmSignupParams = {
            ClientId: ClientId,
            Username: trimmedEmail,
            ConfirmationCode: trimmedCode,
        };

        const command = new ConfirmSignUpCommand(confirmSignupParams);
        const response = await cognitoClient.send(command);
        console.log(response);

        return Response.json(
            {
                error: false,
                message: "Account verified successfully.",
            },
            { status: 200 }
        );
    } catch (error) {
        return Response.json(
            {
                error: true,
                message: error.message || "Account Verification Failed",
            },
            { status: 400 }
        );
    }
};

```

**3. Resend Verification Code API**

**`/app/api/resend-code/route.js`**

```jsx
import { ResendConfirmationCodeCommand } from "@aws-sdk/client-cognito-identity-provider";

import cognitoClient from "@/utils/cognito-config";

export const POST = async (req) => {
    // Grab email from frontend

    const { email } = await req.json();

    // Input Validation

    const trimmedEmail = email.trim();

    if (!trimmedEmail) {
        return Response.json(
            { error: true, message: "Email is required" },
            { status: 400 }
        );
    }

    // Check whether the cognito client is configured or not.

    const ClientId = process.env.WEB_CLIENT_ID;
    if (!ClientId) {
        return Response.json(
            { error: true, message: "Client ID is not configured." },
            { status: 500 }
        );
    }

    // Resend verification code on email address.

    try {
        const resendCodeParams = {
            ClientId: ClientId,
            Username: email,
        };

        const command = new ResendConfirmationCodeCommand(resendCodeParams);
        await cognitoClient.send(command);

        return Response.json(
            {
                error: false,
                message: "Resent verification code successfully.",
            },
            { status: 200 }
        );
    } catch (error) {
        return Response.json(
            {
                error: true,
                message: error.message || "Failed to resend verification code.",
                details: error.toString(),
            },
            { status: 400 }
        );
    }
};
```

**4. Login API**

**`/app/api/login/route.js`**

**4. Logout API**

**`/app/api/logout/route.js`**
