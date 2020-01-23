---
layout: post
theme: jekyll-theme-modernist
title: "Budget Bliss with AWS Lambda and the You Need A Budget API"
date: 2020-01-22
comments: true
crosspost_to_medium: false
excerpt: Combine AWS Lambda and You Need A Budget to get better insight to your finances<p>
---

{: .center}
![]({{ "/resources/2020/01/logos.png" | absolute_url }})

I’ve been a crappy budgeter my whole life. It’s a chore, it’s difficult to manage, and for some reason there is never quite enough money. I’ve tried several products over the years, like Microsoft Money and Mint. I was happy with Mint for a long time, but it never quite fit the bill (forgive the pun). About a year ago I heard about [You Need A Budget](https://www.youneedabudget.com/) (YNAB), and I was immediately intrigued. The YNAB “method” made more sense to me than other tools out there, and it offers a robust and well-documented [API](https://api.youneedabudget.com/). A little poking around in the API docs confirmed that it would be fairly easy to set up a system I had wanted for many years: a [positive feedback loop](https://en.wikipedia.org/wiki/Positive_feedback) of daily budget updates. My vision was to get a text message once or twice a day with the amount of money left in some specific budget categories.

There is a lot of research on feedback loops, both positive and negative, in regards to human behavior. A simple example is a speed trailer placed in a neighborhood. Research shows that the simple act of displaying someone’s speed causes them to slow down. I had the idea that the same principle would apply to budgeting. If my wife and I had a better idea of where things were at budget-wise on a daily basis, we could make smarter spending decisions. I knew this would require some diligence since every transaction has to be assigned a category and approved within YNAB before it is reflected in the budget. This is not too difficult to keep up with since the text message would also serve as a reminder to get into YNAB and categorize any new transactions.

# Enter AWS Lambda

If you’re even remotely paying attention to technology you’ve heard of “serverless” or “Functions as a Service” (FaaS). [AWS Lambda](https://aws.amazon.com/lambda/) is Amazon’s offering in this space, and it has quickly gained adoption due to its ease of use and comprehensive functionality. It’s also cheap - there is no charge when a Lambda function is not in use, and even if I ran my budget alert twice a day that is a max of 62 function executions a month. (Spoiler alert: I’ve had this running for over a month and my AWS bill has gone up by about $0.15. That includes the fees to send text messages via Amazon SNS.)

Using Python to interact with a REST API is easy. [Postman](https://www.getpostman.com/) will [generate the code](https://learning.getpostman.com/docs/postman/sending_api_requests/generate_code_snippets/) needed to make a call in a matter of clicks. From there, it is just a matter of executing the code, and sending the results in a text message. There are many ways to automate text messaging, but I chose to use [Amazon SNS](https://aws.amazon.com/sns/) due to its simple integration with Lambda and low cost.

# The Setup

This example will pull the current amount of money left in one budget category via a Lambda function communicating with the YNAB API, and send it to one or more people via SMS. It’s not too difficult to expand this example to send more budget category balances, if desired. Personally I’ve used AWS for years to host my personal DNS zones via Route53, and I have some home movies stored in Glacier. If you’ve never used AWS before, now is a good time to create an account using [these instructions](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/).

## YNAB

There are few requirements for YNAB, apart from having done the initial setup along with having a budget created. You will need an API key, and to obtain that you must have an account with a username and password. If you are using your Google account (or some other service) to log in to YNAB, you will need to create a password before obtaining your key.

Instructions for obtaining a key, along with API documentation is available at [https://api.youneedabudget.com/](https://api.youneedabudget.com/). To create a key follow these steps:

1. [Sign in to the YNAB web app](https://app.youneedabudget.com/settings) and go to the "My Account" page and then to the "Developer Settings" page.
2. Under the "Personal Access Tokens" section, click "New Token", enter your password and click "Generate" to get an access token.
3. Open a terminal window and run this:
``` 
curl -H "Authorization: Bearer <ACCESS_TOKEN>" https://api.youneedabudget.com/v1/budgets
```
4. You should receive a result beginning with `HTTP/1.1 200 OK` followed by a JSON payload with your budget information.
5. Save your API key in a safe place, like 1Password or LastPass. You will need this key when you are configuring your Lambda function. **This key is essentially the same as your username and password, so don’t share it with anyone**. If you share code on GitHub, remember to remove your key!

Now, use the API explorer to grab an IDs from YNAB to use in your Lambda script. You will need to find the ID for the budget category (or categories) you want to track.

1. From [https://api.youneedabudget.com/](https://api.youneedabudget.com/), click “API Endpoints” in the top right of the page.
2. Scroll down to “Categories” and click the lock icon beside `GET /budgets/{budget_id}/categories`

{: .center}
![]({{ "/resources/2020/01/ynab01.png" | absolute_url }})

{:start="3"}
3. When prompted, paste in your API key, click “Authorize” then close the authorization page. Do not click “Logout”.

{: .center}
![]({{ "/resources/2020/01/ynab02.png" | absolute_url }}){:height="50%" width="50%"}

{:start="4"}
4. Click the `GET /budgets/{budget_id}/categories` line to expand the information for this API endpoint. Click “Try it out”.

{: .center}
![]({{ "/resources/2020/01/ynab03.png" | absolute_url }})

{:start="5"}
5. For budget_id enter `last-used`. If you have multiple budgets this will load the categories for the last budget used. I only have one budget, so this will always default to the correct budget.
6. Click “Execute”

{: .center}
![]({{ "/resources/2020/01/ynab04.png" | absolute_url }})

{:start="7"}
7. Scroll down and examine the response. You will see your budget categories displayed in JSON format. Find the category you want to track, and save the ID listed. Below is a screenshot of one budget category, with the ID partially obfuscated. You should see this pattern repeated for every budget category you have.

{: .center}
![]({{ "/resources/2020/01/ynab05.png" | absolute_url }})

## Amazon SNS

Log into the [AWS Console](https://console.aws.amazon.com/), type “SNS” into the “Find Services” bar, and click “Simple Notification Service” when it comes up. Within SNS, there are two sections to configure: Subscriptions and Topics.

1. Click “Topics”, then “Create Topic”

{: .center}
![]({{ "/resources/2020/01/sns01.png" | absolute_url }})

{:start="2"}
2. Provide a name for your topic as well as a display name, and click “Create topic”. I used “ynab-alerts” for both. Note the “ARN” provided after the topic is created. You will need this value for your Lambda function.

{: .center}
![]({{ "/resources/2020/01/sns02.png" | absolute_url }})

{:start="3"}
3. Click “Subscriptions”, then “Create Subscription”

{: .center}
![]({{ "/resources/2020/01/sns03.png" | absolute_url }})

{:start="4"}
4. Choose the ARN for the topic that you just created, set “Protocol” to “SMS”, and plug in your cell phone number for “Endpoint”. You must follow the format given, e.g. +15555551234. Click create subscription. Amazon will send a confirmation text message to confirm the owner of the phone number entered does indeed wish to participate in the subscription. If not, you would have a very inexpensive way to spam friends (or enemies) with all sorts of texts!

{: .center}
![]({{ "/resources/2020/01/sns04.png" | absolute_url }}){:height="50%" width="50%"}

{:start="5"}
5. Repeat step four for any additional cell phones that you want to send budget updates to.

## AWS Lambda

Click the “Services” dropdown at the top of the page, type “Lambda” and hit enter. This is where we will configure the function used to retrieve information from the YNAB API, and send it to a cell phone via SMS.

1. Click “Create function” on the Lambda dashboard. Leave “Author from scratch” selected, provide a name for your function, choose “Python 3.7” as the runtime, and click “Create function”. Note that the default execution role is sufficient and does not need to be changed. You will now be at the configuration page for your function.

{: .center}
![]({{ "/resources/2020/01/lambda01.png" | absolute_url }})

{:start="2"}
2. Click “Add trigger” and choose “CloudWatch Events”. Click the dropdown under “Rule” and choose “Create a new rule”. Provide a name (e.g. “YNAB-schedule”) and a Schedule expression, and uncheck “Enable trigger” for now. This is a [cron-formatted rule](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html#CronExpressions), so you can specify the times that work best for you to get alerts. I get alerts at noon and 8:00pm, so my expression looks like `cron(0 16,00 * * ? *)`. Once your schedule is set, click “Add”.

{: .center}
![]({{ "/resources/2020/01/lambda02.png" | absolute_url }})

{: .center}
![]({{ "/resources/2020/01/lambda03.png" | absolute_url }})

{:start="3"}
3. You should be back on the Lambda function configuration page. If “CloudWatch Events” is still highlighted in the designer, click on the Lambda function name so that the code editor is displayed.

{: .center}
![]({{ "/resources/2020/01/lambda04.png" | absolute_url }})

{:start="4"}
4. Scroll down to view the built-in code editor. In another tab, open the example code from this [GitHub Gist](https://gist.github.com/shamsway/49e2a7f32a18cc9563c50cd1ba59f2ae). Copy that code and paste it into the Lambda code editor. Replace the placeholders for API Key (line 6), Budget category (line 7), and SNS Topic ARN (line 31). The editor will save code automatically, but to be safe click File->Save.

{: .center}
![]({{ "/resources/2020/01/lambda05.png" | absolute_url }})

{:start="5"}
5. At the top of the page, click the dropdown beside the “Test” button and choose “Configure test events”. Provide an event name (e.g. “Test”) and click “Create”. Now click the “Test” button, and if everything is working as expected, you will receive a text with the amount of money left in your budget category!

{: .center}
![]({{ "/resources/2020/01/lambda08.png" | absolute_url }})

{: .center}
![]({{ "/resources/2020/01/lambda06.png" | absolute_url }})

{:start="6"}
6. Feel free to adjust the script to change wording or check additional categories based on the steps outlined. It will require some Python knowhow to adjust the existing script and add the logic and outputs for additional categories, but I’m happy to help you if needed. Once you are satisfied with the alert you’re getting when the function runs, click on “CloudWatch Events” in the “Designer” section, then click the slider to enable your schedule. You will now receive texts matching your defined schedule.

{: .center}
![]({{ "/resources/2020/01/lambda07.png" | absolute_url }})

# Final Thoughts

Here’s a screenshot of the text that my wife and I get twice a day. I’m tracking four different categories, but the code is a bit hacky so I’ve opted not to share it.

{: .center}
![]({{ "/resources/2020/01/ynab-alert.png" | absolute_url }}){:height="50%" width="50%"}

Here is my monthly AWS bill with the relevant services highlighted. Clearly this is easy to fit into the budget.

{: .center}
![]({{ "/resources/2020/01/aws-bill.png" | absolute_url }})

I hope this is a helpful exercise for other YNAB users out there, or at least an example showing the power of APIs and serverless functions. There are many APIs available for consumption, so with a little imagination you can build all sorts of useful tools by following this example. I chose not to share the script I’m using to monitor multiple categories because it is fairly ugly and could use some enhancements. Ultimately I’d like to trigger this function with an API call, and pass in the categories to track via a JSON payload. This would allow me to schedule the same twice-daily messages, but I could also use the script to check categories from Alexa or some other method.

Did I miss anything? Please comment below with questions, or find me on twitter at [@NetworkBrouhaha](https://twitter.com/NetworkBrouhaha).