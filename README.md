## Link Checker Example
We are going to make an app that checks a website to see if it has any dead links on the page.  The website will be very simple.  For a given url, it will get the contents of the page, follow all of the links on that page, and then show the results of the check.

First, create the app:

```
rails new LinkChecker
```

Next edit our routes in ```config/routes.rb```:

```
LinkChecker::Application.routes.draw do
  root 'sites#new'
  get '/sites/new', to: 'sites#new', as: 'new_site'
  post '/sites', to: 'sites#create', as: 'sites'
  get '/sites/:id', to: 'sites#show', as: 'site'
  get '/linkfarm', to: 'sites#linkfarm'
end
```

Add a controller:

```
rails generate controller sites new create show --no-test-framework --no-helper
```
Next, get rid of the files we don't need:

```
rm app/assets/javascripts/sites.js.coffee app/assets/stylesheets/sites.css.scss
rm app/views/sites/create.html.erb
```

Also, open up ```config/routes.rb``` and remove the routes that were created that we do not need.  Your routes.rb file should look just like the routes file above.

Then create the models. In this project we are modeling two things.   There is a website, or a site.  A site is an html page that we want to check for dead links.  A site will have many urls on the site.  So there is a 1 to Many relationship.

```
rails generate model site url:string --no-test-framework
```

```
rails g model link url:string http_response:integer site:belongs_to --no-test-framework
```

Next the models need to know about how they are associated.  In ```app/models/link.rb``` add:

```
belongs_to :site
```

And in ```app/models/site.rb``` add:

```
has_many :links
```

Next, add all the necessary gems to the ```Gemfile```:

```
gem 'pg'

gem 'nokogiri'

gem 'typhoeus'

gem 'dotenv-rails', :groups => [:development, :test]

gem 'rails_12factor', group: :production
```

Also remember to edit ```config/database.yml``` to use postgresql and edit ```config/initializers/secret_token.rb``` to use environment variables instead of exposing the secret keys.  Store the secret key in the .env file instead.

### Link Checker Algorithm

This section provides a brief description of what the link checker will be donig in the controller create method.  Here is an algorithm in puesdo code:

```
Get the parameter for the url to check
Create a site record in the database (Save the url)
Use Nokogiri to get the page and parse the contents
Use Nokogiri to get all anchor tags
For each anchor tag, do:
   Get the href from the anchor tag
   Make sure the href starts with http, https or is a relative path
   Use Typhoeus to get the linked page
   Create a new recored in the DB for the link (Saves link url, and http response code)
end

redirect to the show page for the site
```

### Main Link Checker Code in Create

```
def create
  require 'open-uri'
  url = params.require(:site)[:url]
  site = Site.create(url: url)
  
  contents = Nokogiri::HTML(open(site.url))
  contents.css('a').each do |link|
    link_href = link.attributes['href'].value
    if (link_href.starts_with? '/')
      link_href = site.url + link_href
    end
   
    if (link_href.starts_with? 'http://', 'https://')
      response = Typhoeus.get(link_href)

      site.links.create(url: link_href, http_response: response.response_code)
    end  
  end
  
  redirect_to site_path(site)
end
```

### Exercise

__Discussion__ What is wrong with the performance of the code?  How could it be improved?

### Moving Work to Sidekiq

Since the above logic is very slow, let's move it to a sidekiq task.  First, add a directory called ```app/workers```.  Next, create a file in the workers directory called ```links_worker.rb```.  Now that we have the worker file, lets make some changes to ```app/controllers/sites_controller.rb```.  In the create method, remove all the code that deals with fetching data from a url.  Instead, just have create make a new site in the database and then preform the async task that we want.

```
def create
  url = params.require(:site)[:url]
  site = Site.create(url: url)
  LinksWorker.perform_async(site.id)
  redirect_to site_path(site)
end
```

It's important to note that the perform_async method uses the site.id to identify the work that needs to be done, not the ActiveRecord object itself.  All of the data that is passed to perform async actually has to be written to Redis.  If we pass too complicated of an object, Redis may have problems getting the correct data back.  It's safer and easier to pass the id itself and then look up the object in the worker.

Here is the code for the worker that is mainly copied and pasted from the create method.  The code is in ```app/workers/links_worker.rb```:

```
class LinksWorker
  include Sidekiq::Worker

  def perform(site_id)
    require 'open-uri'
    site = Site.find(site_id)

    contents = Nokogiri::HTML(open(site.url))
    contents.css('a').each do |link|
      link_href = link.attributes['href'].value
      if (link_href.starts_with? '/')
        link_href = site.url + link_href
      end

      if (link_href.starts_with? 'http://', 'https://')
        response = Typhoeus.get(link_href)

        site.links.create(url: link_href, http_response: response.response_code)
      end  
    end
  end
end
```

Now the majority of the work is done in the background with the ```LinksWorker``` class.  

### Testing Workers

Testing a worker is fairly simple because we can run a worker without using sidekiq at all.  To manually test, open up `rails c` and call the worker.  In the LinkWorker example, the code would be:

```
site = Site.first
LinksWorker.new.perform(site.id)
```

You can also add a spec for the worker.  For the link worker spec, the file would be `spec/workers/links_worker_spec.rb`.  Here is the code:

```
require 'spec_helper'

describe "LinksWorker" do
	describe "Perform check url" do
		let(:data){ {url: "https://www.google.com"} }
		it "Gets a page and puts all the urls in the database" do
			site = Site.create(data)
			LinksWorker.new.perform(site.id)
			site.links.should_not be_nil
			site.links.length.should > 0
		end
	end
end
```
