You get it by signing in with Firebase and using the ID token from the Firebase Auth API.

The guide’s flow is:

1. Set your Firebase web API key
    
    - Put the same public web key into your environment as FIREBASE_WEB_API_KEY.
2. Send a sign-in request
    
    - Use the Firebase endpoint:
    - [https://identitytoolkit.googleapis.com/v1/accounts:signInWithPassword?key=YOUR_FIREBASE_WEB_API_KEY](vscode-file://vscode-app/Applications/Visual%20Studio%20Code%20ARM.app/Contents/Resources/app/out/vs/code/electron-browser/workbench/workbench.html)
3. Send this JSON body
    
    - email
    - password
    - returnSecureToken: true
4. Read the idToken from the response
    
    - That value is the Firebase ID token.
    - In the guide, this is assigned to FIREBASE_TOKEN.
5. Use it as a Bearer token
    
    - Send it in the Authorization header:
    - Authorization: Bearer <idToken>

Example shape:

- POST to Firebase sign-in endpoint
- parse .idToken from the JSON response
- then call your backend with:
    - Authorization: Bearer <that idToken>

Important:

- This is a Firebase ID token, not a password.
- It is only valid for authentication and should not be shared publicly.