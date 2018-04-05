# Adventures with AWS Secrets Manager

[AWS Secrets Manager](https://aws.amazon.com/blogs/aws/aws-secrets-manager-store-distribute-and-rotate-credentials-securely/) was officially announced to be GA yesterday. Secure password rotation is just about an issue everywhere, so I very much looked forward to playing around with this.

**General FAQ**

Q: What does it do?
A: It stores and rotates credentials

Q: What sort of credentials?
A: Database logins, various secrets, or even complete custom ones

Q: How does it work?
A: After you configure the required information, a lambda function is automatically created that contains the scripts to rotate the credentials.

Let's dig in.

## Setting up Database Password Rotation

The general idea is what we want to store two secrets. One is the master login for the database. Then a second login, whether it's an user or service account, that needs to be rotated.

### Create the static master secret

1. Log into AWS Console and search for Secrets Manager. Should pop up right up.

    Click the "Store New Secret" button to get started.

    ![](/screenshots/0005/01.png)

2. We are then taken to the first screen of creating a secret. Two things are highlighted here.

    ![](/screenshots/0005/02.png)

    At the top you can select whether it's a RDS database or another database that you have to manually put the information for. In our example we're using a RDS database. Simply put in your master login and password, select the RDS database and click next.

    If you choose a custom database here, not only you have to provide the host and port information, you'll later have to adjust the lambda function's security group to make sure the database is accessible by that function.

    Another thing to watch out is the encryption key. If you choose anything other than the default, you'll later have to adjust the IAM role of the lambda function to make sure the KMS key is accessible.

3. The next screen we simply have the name and description of the secret. Put in something appropriate and move on.

    ![](/screenshots/0005/03.png)

4. Then on this screen, we can ignore for the master secret. Ideally the master secret should be rotated too. Because we provision our databases with Terraform and Vault, I need to make sure the new credentials is fed back into our system, and I haven't done that yet. So in this example, we won't rotate the master.

    Leave it disabled and move on.

5. Finally, review all settings and create the secret.

### Create the secret getting rotated

With the master secret stored, we can now create the service login that we want rotated.

Go through the same steps as before. Put in the current login for the user/service account. Keep in mind that the account should already exists on the database, and has access to the specified schema.

**If you are using MySQL, please make sure that your login length is 10 characters or less**. The reason is that Secrets Manager will alternate between `<login>` and `<login>_clone` as username, and most MySQL versions only supported 16 characters in username.

Only the screen below is different.

![](/screenshots/0005/04.png)

We want to use the previous master secret that we created to rotate this one. So the select the highlighted option, and pick the new master secret that we just created.

Follow through and finish creating this secret. You should be greeted by a helpful message at the top of the page.

![](/screenshots/0005/05.png)

#### Adjust IAM role

If you wait a minute or two as the message suggested, you'll see a different message appearing.

![](/screenshots/0005/06.png)

When you created the secret in the last step, AWS automatically created the IAM role and the Lambda function required to rotate the secret. However, that IAM role also needed access to read your master secret, otherwise it cannot log into the database to rotate the credential.

1. From that message, copy the ARN of the master secret, you'll need it later.

2. Click on the "role" link on the message. It will open a new tab that takes you directly to the IAM role. Here we need to make some edits.

3. ![](/screenshots/0005/07.png)

    Here we see two inline policies that ends in -0 and -1. Expand the -1, and click "Edit Policy"

4. ![](/screenshots/0005/08.png)

    In this next screen, we click "Add additional permissions" on the right side.

5. Now you should have a new section to fill in. Fill it in like below:

    - Service: Secrets Manager
    - Actions: GetSecretValue
    - Resources: paste in the ARN that you copied from step 1 in this subsection

    ![](/screenshots/0005/09.png)

6. Finish setting up by clicking through the rest of the prompts.

### Test it out

Back to the Secrets Management page and click on the secret that is meant to be rotated. You should find a "Rotate Secret Immediately" button. You might get an error clicking it the first time saying a previous rotation needs to be completed. But that should only happen for the first time.

You can grab the Lambda function's ARN and look up the logs in Cloudwatch. Should tell you exactly what's going on.

If you rotate it a couple times in succession, you'll see the username rotating between `<username>` and `<username>_clone`.
