# `iphab` -- tools to integrate Phabricator into the IETF workflow

IETF involves a lot of document review, and this is doubly true for being an
Area Director. `iphab` merges the IETF workflow with the popular
[Phabricator](https://phacility.com/phabricator/) platform.

Major features:

- Automatically maintain Phabricator "revisions" (Phabricator's term for change
  lists/pull requests/etc.) for each Internet-Draft, with a new version for each
  draft that's published.

- Assign reviewers to revisions based on the IESG agenda.

- Format reviews so they are suitable for mailing.

- Post reviewed revisions on the IETF Ballot tool.

The intent is that eventually much of this will run on a server and keep
Phabricator in sync, but at the moment it's kind of manual and needs one person
to drive it, though multiple people can review. Email me if you want to share
the existing instance.

Differences to [Ekr's original version](https://github.com/ekr/iphab):

- converted to Python 3

- use Phabricator URL from the Arcanist config file in the generated reviews

- removed apparently obsolete file

## Setup

1. Install the Phabricator [Arcanist
   tool](https://secure.phabricator.com/book/phabricator/article/arcanist/).

2. Check out the dummy IETF review at https://github.com/ekr/ietf-review

3. Create the `.arcconfig` file in the `ietf-review` repo. I use:

   ``` json
   {
       "phabricator.uri" : "https://phab.eggert.org/"
   }
   ```

   You should substitute the URL of your Phabricator instance.
  
4. Initialize `arcanist` in the `ietf-review` repo. You'll want to create a bot
   account in Phabricator other than the one you use for actual review, because
   you can't really review your own revisions. You'll need to create an API
   token in Phabricator for the bot account, run

   ``` shell
   arc install-certificate
   ```
  
   in the `ietf-review` repo, and paste the API token into the shell when asked.

5. Create the `.iphab.json` file. It lives in the working directory (the parent
   of `ietf-review`) and looks like:

   ``` json
   {
       "review-dir" : "<where you want downloaded reviews to go>",
       "reviewer": "<your phabricator username>",
       "apikey": "<your datatracker API key>"
   }
   ```

   If you're not an AD you won't need the datatracker API key.

6. In your Phabricator configuration, you need to set the
   `differential.require-test-plan-field` option to `false`, otherwise the draft
   uploads into Phabricator will fail.

   In order to let authors access the HTML version of the reviews, you also want
   to set the `phabricator.application-settings` option to

   ``` json
   {
       "PHID-APPS-PhabricatorDifferentialApplication": {
           "policy": {
               "differential.default.view": "public"
           }
       }
   }
   ```

## Keeping Phabricator in Sync

Once you have things set up, you need to periodically update Phabricator with
the current drafts. `iphab` will make a local copy of all drafts with `rsync`
and then will create a new revision for each draft it doesn't know about or
otherwise update the revision for each new draft.

This is done with:

``` shell
iphab update-drafts
```

It will take hours the first time, as it creates revisions for every ID, and
minutes thereafter. You might want to disable email notifications in Phabricator
(Settings -> Email Delivery) before running this the first time, or you will
lots of emails! Re-enable them after the first run.

## Managing the IESG Agenda

If you are an AD, `iphab` can automatically assign you reviews based on the
current agenda. This is done with:

``` shell
iphab update-agenda
```

Any draft on the agenda will be assigned to the `reviewer` value in your
`.iphab.json` config file. You will be assigned as a "blocking" reviewer for
standards track and as a regular reviewer for non-standards track. This way you
can see a dashboard in Phabricator.

## Reviewing

You review in the usual way on Phabricator, filing inline comments and overall
comments.

You can download a formatted review to mail to somebody using:

``` shell
iphab download-review <draft-name>
```

The review gets stuffed in `CONFIG["review-dir"]/<draft-name>-rev.txt`.

## IESG Balloting

You can also auto-ballot. `iphab` will take your review of the latest revision,
turn it into an IESG ballot, and upload.  You will first need to have a
[datatracker API key](https://datatracker.ietf.org/accounts/apikey), which goes
in the `.iphab.json` config file.

For AD balloting, the two states: "Needs-revision" and "Accepted" have the
special meaning of DISCUSS and NO-OBJECTION. Any other state will be kicked out.

For DISCUSS ballots, the top comment and any comment marked "IMPORTANT:" will be
turned into the Discuss portion of the ballot, with every other inline comment
turned into the Comment portion.

For NO-OBJECTION, the top comment, IMPORTANT comments, and other comments each
get their own sections in a single text field.

Right now this is regrettably not totally automated, so once you have done your
review, you need to do:

``` shell
iphab ballot <draft-name>
```

## Clearing your Review Queue

If you occasionally don't review things you are supposed to, your review queue
can get cluttered. `iphab clear-requests` will clear anything you haven't
reviewed.
