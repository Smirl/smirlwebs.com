---
title: "Creating a dynamic review system for a static Hugo site"
date: 2021-09-23T12:58:17+01:00
image: uploads/blog-thumb-01.png
feature_image: uploads/blog-post-01.png
author: Alex Williams
---

Using a Golang backend, we can create a rich form system that sends emails,
raises pull requests, and does some basic spam protection.
{.intro}
<!--more-->

Static website generators are great. However, the biggest drawback is their also
their biggest strength, they are static. Often we need forms to collect
information from our users, most common is the trusty contact form. Things like
[formspree.io] are good for this but require a subscription. This is totally
fine for a commercial website as the plans are often affordable, but for hobby
sites it isn't ideal. Similarly, for dynamic content we can use [Staticman]
which raises pull requests with new content. The work here is heavily inspired
by [Staticman].

I, like most developers that make websites as a hobby, completely over engineer
my sites and use them as a playground to learn new things. For example I deploy
my sites to a Kubernetes cluster in digitalocean, because I can not because I
should. Which means that running a small `golang` server to act as a form
backend, and also host the statically generated html wouldn't be too hard.

The website that I was working on, [High Heath Farm Cattery], has 3 forms:

- a simple contact form,
- a booking form registering interest in a stay for you cat,
- and a review form, for letting users share their experience of using the
  cattery

## The Review Form HTTP Handler

Let's go over the handler for the review form, with the high level steps that we
take.

For the review form, a `POST` request is set to the golang backend where it is
parsed into an instance of a struct using [gorilla/schema]. Fields such as
`Date` which are not added on the client side are added here too. Some basic
form validation is done checking that certain fields are not filled by bots, and
checking the recaptcha token is valid.

The comment content markdown is then generated, and a pull request raised in
`CreateComment`. We go into details of this in the next section.

Finally, we send emails to the customer and the business owner using the Gmail
SDK. Sending the email on behalf of the business owner is very easy, however we
can't fake an email on behalf a customer. For this we instead add an email into
the business owners inbox using the `client.Users.Messages.Insert` API.

The handlers for the other forms are very similar, validating data and sending
emails. `HandleCommentForm` therefore handles [formspree.io] and [Staticman]
responsibilities.

```golang
func HandleCommentForm(c AppContext, w http.ResponseWriter, r *http.Request) {
	if err := r.ParseForm(); err != nil {
		log.Printf("Unable to parse form: %v", err)
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	var form Comment
	if err := c.Decoder.Decode(&form, r.Form); err != nil {
		log.Printf("Unable to decode form: %v", err)
		w.WriteHeader(http.StatusBadRequest)
		return
	}
	form.Date = time.Now()

	if err := ValidateForm(c.Recaptcha, &form); err != nil {
		HandleFormError(w, r, &form, err, "/comments/failure/")
		return
	}

	if err := CreateComment(r.Context(), c.GithubClient, &form); err != nil {
		log.Printf("Error creating comment pull request: %v", err)
		HandleFormError(w, r, &form, err, "/comments/failure/")
		return
	}

	if err := SendMessages(c.GmailClient, &form); err != nil {
		HandleFormError(w, r, &form, err, "/comments/failure/")
		return
	}

	http.Redirect(w, r, "/comments/success/", http.StatusFound)
}
```

## The CreateComment Function

This function takes the parsed form and an instance of the Github Client. We
need to get the SHA of the commit at the tip of master so we can create a new
branch from that point. The branch name is the same as the slug used in the
content, i.e. `20210102-150405`.

```golang

func CreateComment(ctx context.Context, client *github.Client, comment *Comment) error {
	// Slug used for filename and branch name
	slug := comment.Date.Format("20060102-150405")

	// Get SHA of master branch
	ref, _, err := client.Git.GetRef(ctx, "smirl", "highheath", "heads/master")
	if err != nil {
		return err
	}

	// Create branch from lastest master SHA
	newRef := fmt.Sprintf("refs/heads/%s", slug)
	newRefObj := github.Reference{Ref: &newRef, Object: &github.GitObject{SHA: ref.Object.SHA}}
	if _, _, err := client.Git.CreateRef(ctx, "smirl", "highheath", &newRefObj); err != nil {
		return err
	}

	// Create file commit
	path := fmt.Sprintf("content/comments/%s.md", slug)
	message := fmt.Sprintf("🤖💬 New comment from %s", comment.GetName())
	opts := &github.RepositoryContentFileOptions{
		Content: comment.GetFileContent(),
		Branch:  &slug,
		Message: &message,
	}
	if _, _, err := client.Repositories.CreateFile(ctx, "smirl", "highheath", path, opts); err != nil {
		return err
	}

	// Create Pull Request
	base := "master"
	body := fmt.Sprintf("New comment:\n\n**Name**\n%s\n**Message**\n%s\n", comment.Name, comment.Message)
	newPR := github.NewPullRequest{Title: &message, Head: &slug, Base: &base, Body: &body}
	pr, _, err := client.PullRequests.Create(ctx, "smirl", "highheath", &newPR)
	if err != nil {
		return err
	}

	// Label the Pull Request
	labels := []string{"comment", "patch"}
	assignee := "smirl"
	updatedPR := github.IssueRequest{Labels: &labels, Assignee: &assignee}
	if _, _, err := client.Issues.Edit(ctx, "smirl", "highheath", *pr.Number, &updatedPR); err != nil {
		return err
	}
	return nil
}
```

## Conclusion



[formspree.io]: https://formspree.io/
[Staticman]: https://staticman.net/
[High Heath Farm Cattery]: https://www.highheathcattery.co.uk/
[gorilla/schema]: https://github.com/gorilla/schema