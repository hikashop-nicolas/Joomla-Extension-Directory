# Sign in with the Joomla Identity Portal

Design proposal for issue #20 (Integration - Identity Portal). Nothing here is built yet. The
open questions at the end need answers from the Joomla operations team before implementation.

## Goal

A person signs in to the JED with their Joomla Identity Portal account, and their existing JED
account, extensions, reviews and favourites follow them.

## What is known

Checked on 2026-10-19 against the live sites and the public code in
https://github.com/joomla/volunteers.joomla.org. There is no written specification that I could find.

- The Identity Portal is https://identity.joomla.org, a Joomla site. It describes itself as a
  single sign-on provider and the place where a person manages their profile and consents.
- Connected today: the Identity Portal itself, Volunteers, Downloads, Guide, the Developer Network
  and Mattermost. The JED is not listed.
- The protocol is SAML 2.0. The login pages of Downloads and Volunteers redirect to
  https://identity.joomla.org/sso/module.php/saml/idp/singleSignOnService with a SAMLRequest and a
  RelayState holding the page to return to. That path is SimpleSAMLphp's.
- The Volunteers repository bundles com_sso 1.3.0 (RolandD Cyber Produksi, GPLv3, 2021), which wraps
  SimpleSAMLphp 1.19 and was written for Joomla 3.5. That site runs PHP 7.4.
- On a connected site the local username is the person's GUID. The client plugin makes name, username,
  email and password read only.
- The portal also pushes profile changes to each site. The request is `?joomlaidp=1` with `data`
  (JSON: guid, task, userData) and `hash` (base64 HMAC-SHA512 of `data`, keyed with a 36 character
  per-site API key). The site updates the local user and fires an `onProcessIdentity` event for
  site specific data. Volunteers uses it for the volunteer profile.
- Extra profile data is shared with a site only after the person gives that site a consent. Withdrawing
  a consent makes the portal ask the site to redact and anonymise the data.

## What is not known

The IDP metadata URL, the attributes it releases, the NameID format, whether it signs assertions,
responses or both, how sign out works, and how a new site is registered. These are in the questions below.

## Design

### Pieces

1. **plg_authentication_joomlaidentity**, a SAML 2.0 service provider.
   - Login: redirect to the IDP with the page to return to, handle the response at an assertion
     consumer URL, find or create the user, log them in.
   - Optional single logout.
   - Settings: IDP metadata URL or the entity id, sign-on URL, signing certificate, our entity id, our
     key pair, a switch to keep the password form, and the attribute names.
2. **plg_user_joomlaidentity** (or part of 1): the link table and the user form changes.
3. **Profile endpoint**: receives the portal's signed push, same contract as the client plugin the
   other sites use. Optional, only needed if the JED wants profile data or consent changes from the portal.
4. **Link table** `#__jed_identity_links`: `user_id`, `guid` (unique), `email_at_link`, `linked`, `source`
   (first login, import, admin).

### SAML library

com_sso cannot be reused, since it targets Joomla 3 and PHP 7. Candidates to evaluate before choosing:
SimpleSAMLphp 2 as a library, OneLogin php-saml, and simplesamlphp/saml2. The choice is made after
the IDP answers (signing, binding, logout), and after checking each one's PHP 8.3 to 8.5 support and
maintenance. Whatever is chosen is installed through Composer into the plugin, and covered by the
existing audit job.

### Matching a person to a JED user

At assertion time, in order:

1. A link row for the GUID exists: log in as that user.
2. The assertion carries a verified email that matches exactly one JED user: link that user to the
   GUID, then log in. Never link on an unverified email, and never to a blocked or unactivated user.
3. The email matches more than one JED user: do not guess. Show a page that asks the person to
   choose and prove control of the account (the JED password, or a mailed link), and log it.
4. No match: create a user (username is the GUID, as on the other sites), put it in the Registered
   group, and apply the normal JED registration defaults.

The JED user id never changes, so extensions (owner, maintainers), reviews, favourites, tickets and
transfers stay attached. The migrated JED data keeps its Joomla user ids.

### Name and email

If the portal owns them, the JED makes them read only for linked users, as the other sites do. A
change of email at the portal updates the JED user through the profile endpoint or at the next login.
If the new email collides with another JED user, keep the old email and log it for an admin.

### Fallback

Keep password login for existing users until the operations team says otherwise. A switch in the
plugin hides it later. Backend login stays password based unless the team wants the portal there too.

### Privacy

The JED already has export and erasure services. A consent withdrawal from the portal should run the
same erasure path as a privacy request for that user, so there is one code path. Reviews are public
content, so what is redacted on withdrawal needs a decision (question 8).

### Security

- Validate the assertion signature against the pinned IDP certificate, the audience, `NotOnOrAfter`,
  and `InResponseTo` against a request id stored in the session. Reject unsigned responses.
- RelayState is only followed if it is a path on this site (no open redirect).
- Compare the HMAC with `hash_equals`. The existing client plugin uses `!==`. Also ask the portal
  to send a timestamp in the payload so a captured push cannot be replayed.
- Rate limit the assertion consumer and the profile endpoint through the existing rate limit plugin.
- Keep the SP private key outside the web root, or at least readable by the web server only.

### Tests

- Unit tests for the matching rules above (one match, many matches, unverified email, blocked user,
  email change collision).
- A SAML test IdP in CI (a small container such as a SimpleSAMLphp or a mock IdP image) so Cypress can run the
  whole login: first login creates a user, second login reuses it, an existing JED user is linked by email,
  a tampered response is refused, a bad RelayState is ignored.
- A test for the profile endpoint: good hash, bad hash, replay.

## Phases

1. Answers to the questions below, and a test registration of the JED as a service provider with the IDP.
2. Login plugin with matching and the link table, behind a switch, tested against the mock IdP.
3. Profile endpoint and consent handling.
4. Pre-linking existing users by email before the migration, then the switch on the new JED.

## Open questions for the Joomla operations team

1. Is the JED meant to be a connected site, and who registers it? What does the portal need from us
   (entity id, assertion consumer URL, certificate, logo, privacy contact)?
2. Is there a specification or a private repository for the IDP side? Where is the IDP metadata?
3. Which attributes does the IDP release (GUID, name, email, email verified flag, anything else), and
   which NameID format?
4. Are assertions, responses or both signed, and encrypted? Is single logout supported, and is sign
   out of the portal expected to sign out of the JED?
5. Are portal emails verified, and unique across portal accounts? Can one portal account have several
   emails?
6. How should existing JED accounts be linked? Our proposal is by verified email at first login.
   Is there a better key, for example a list of old JED users that the portal already knows?
7. Should password login on the JED stay as a fallback, and for how long?
8. Does the JED need its own consent type in the portal? What should happen to public reviews and
   extension listings when a person withdraws it: anonymise the author, or keep them as they are?
9. Is the profile push needed for the JED, or is login enough?
10. Should the backend login use the portal as well?
11. Is there a staging portal we can register against, and a contact for it?
