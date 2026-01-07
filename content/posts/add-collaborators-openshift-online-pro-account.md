---
title: "How to Add Collaborators to Your OpenShift Online Pro Account"
date: 2018-01-17T13:56:27
draft: false
slug: add-collaborators-openshift-online-pro-account
categories:
  - Uncategorized
wordpress_id: 160
---
*(**Note: **This post originally appeared on the [Red Hat OpenShift Blog](https://blog.openshift.com/add-collaborators-openshift-online-pro-account/))*

OpenShift Online recently made a new feature available to Pro accounts: Collaboration. Collaboration allows a Pro account user to provision cluster account access for other users, called *collaborators*. These collaborators have normal access to the same cluster as the Pro account (without any resource quotas or ability to create new projects) and thus can be granted permissions to work on projects owned by the Pro account.

Every Pro account has the ability to add up to 50 collaborators to their subscription. Collaborators are free cluster accounts provisioned by the Pro subscription owner. In addition, one collaborator account can be used for collaboration with multiple different Pro subscriptions (this is important to note for security reasons because, as we’ll discuss later in more detail, simply removing a collaborator from your subscription does not necessarily remove them from all cluster access).

Collaboration will greatly improve workflows for teams choosing to host their projects on OpenShift Online, as previously the only way to have multiple cluster accounts was to sign up for an additional Pro subscription for each account. Along with saving money, Collaboration also saves time by creating a simpler method for creating new accounts.

### Adding Collaborators to Your Subscription

To get started, first you will need an OpenShift Online Pro account (if that wasn’t obvious already). From there, each user you wish to add as a collaborator will need to create a free account at [developers.redhat.com](https://developers.redhat.com/).

![](https://blog.openshift.com/wp-content/uploads/register.png)

![](https://blog.openshift.com/wp-content/uploads/signup.png)

Once your collaborator has confirmed their Red Hat Developers account, you can add them to your subscription. First, have the user sign into [developers.redhat.com](https://developers.redhat.com) and click on their name in the upper-right corner to access their account details. Make note of the “Red Hat Login ID” on this page, as it is the username you will be required to enter in order to associate the Collaborator with your subscription.

![](https://blog.openshift.com/wp-content/uploads/username.png)

Now sign in to [manage.openshift.com](https://manage.openshift.com/) and click on **Manage Subscription** under the cluster you wish to add them to.

![](https://blog.openshift.com/wp-content/uploads/manage.png)

Once you are in the subscription management console, click the new **Manage** link beside *Collaborators*, which will bring you to the collaborators page.

![](https://blog.openshift.com/wp-content/uploads/UPDATED.png)

On the collaborator page, enter the Red Hat Login ID for the user in the username field and click **Add collaborator**

![](https://blog.openshift.com/wp-content/uploads/adduser.png)

You should now see the user listed under your Collaborators, as well as the time that they were added and an option to remove them from your subscription.

![](https://blog.openshift.com/wp-content/uploads/success.png)

Note that this does not automatically grant the user any access to your projects. Access will need to be granted manually by the project owner, most likely you, using OpenShift policy commands.

### Granting Collaborators Project Access

Once a collaborator account has been provisioned on the cluster, they will have the ability to be given permissions to access any project on the cluster. They can also use the same account to collaborate under multiple different subscriptions.

Note that while this does mean that a collaborator provisioned by *Subscription A* can be given access to any other *Subscription B* ’s projects without counting toward the collaborator limit for *Subscription B*, if the collaborator is removed from *Subscription A* at any time that collaborator’s cluster account will be deprovisioned (and they will lose all access to *Subscription B* ’s projects).

Thus, the only way to *guarantee persistence* of collaborator accounts for as long as your subscription is active is to add them to your subscription through the “Manage Collaborators” page. Simply granting a user project access is not enough to make them permanent, but it is required for them to see your projects.

There are two ways to grant project access to a user on the cluster:

#### Granting Access with the CLI

One way is to log in to the cluster through the CLI using your access token and use `oc policy add-role-to-user` to give the user a role using the same username listed on the Collaboration page:
~$ oc login https://api.openshift.com --token=

Logged into "https://api.openshift.com:443" as "mdame" using the token provided.

You have one project on this server: "mdame-collab"

Using project "mdame-collab".

~$ oc policy add-role-to-user view collaborator-1234

role "view" added: "collaborator-1234"

*(This example grants “view” access to the project for user “collaborator-1234”. [Learn more about access roles here.](https://docs.openshift.com/online/architecture/additional_concepts/authorization.html))*

#### Using the Web Console

Another method is to use the OpenShift Web Console (by clicking the **Open Web Console** link in the subscription manager) and navigating to **Resources > Membership**:

![](https://blog.openshift.com/wp-content/uploads/membership-ui.png)

On the next page, click **Edit Membership**

![](https://blog.openshift.com/wp-content/uploads/edit-membership.png)

From here you can add the collaborator by their username, select the appropriate role, and click **Add**. When you’re finished, click **Done Editing**

![](https://blog.openshift.com/wp-content/uploads/add-user.png)

Now, when the user signs in to [manage.openshift.com](https://manage.openshift.com/), they will see a card to log in to the web console for the same cluster as your subscription and, if they’ve been granted it, will have access to your projects on the cluster just like any other user.

![](https://blog.openshift.com/wp-content/uploads/collab-card.png)

### Removing Collaborators

If at any time you wish to remove the user as a Collaborator from your subscription, you can do so on the same Collaboration page you used to add them (either by checking multiple collaborators and using the **Remove Selected** button to batch-remove multiple users, or by clicking the red **Remove** button next to each collaborator to remove them one-by-one).

It is important to note, however, that *removing a collaborator from your account will not automatically remove any access roles you have assigned the user in your projects*. These will need to be manually deleted (similar to how they were created) or the user may still have access to your projects. Because one user can be a collaborator on the same cluster for multiple subscriptions, simply removing them from your subscription may not remove their cluster access.

As part of the team that developed this feature, we are all very excited about the launch of Collaboration in OpenShift Online. This feature will help teams and organizations work together with the benefits of a hosted OpenShift cluster, and we hope you have as much fun using it as we did building it.
