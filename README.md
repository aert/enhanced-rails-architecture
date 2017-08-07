# Enhanced Ruby on Rails Architecture

Ruby on Rails is an awesome framework beloved by many. But as your application grows you get a questions very soon, where to put this piece of code or that piece over there.
From some point it is on place to introduce new patterns for common scenarios. I will introduce few of them here.

We will start with **concerns** which I guess most Rails developers know well. And many don't consider it as a good practice. **Helpers** are the same case.  
For complex forms composed of more models we will have a look on **form objects**.  
We will talk about **decorators** as well which are presented by gem **Draper**
(https://github.com/drapergem/draper).
Very close to decorators are **policies**, for which we will use **Pundit** (https://github.com/elabs/pundit).  
Many other languages have **publisher-listener** mechanisms built in. In ruby you can easily achieve the same functionality by **Wisper** (https://github.com/krisleech/wisper).
And we will close it with **services**, probably the most hackneyed pattern of those a bit advanced ones.

## Folder structure

To keep track of all those new classes that will appear in the project it is good to come with new folders. The folders are not fixed, but the following structure makes sense:

app/  
&nbsp;&nbsp;&nbsp;&nbsp;&bull; controllers  
&nbsp;&nbsp;&nbsp;&nbsp;&bull; [decorators](#decorators)  
&nbsp;&nbsp;&nbsp;&nbsp;&bull; [helpers](#helpers)  
&nbsp;&nbsp;&nbsp;&nbsp;&bull; [listeners](#publishers-and-listeners)  
&nbsp;&nbsp;&nbsp;&nbsp;&bull; models/  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&bull; [concerns](#concerns)  
&nbsp;&nbsp;&nbsp;&nbsp;&bull; [policies](#policies)  
&nbsp;&nbsp;&nbsp;&nbsp;&bull; [publishers](#publishers-and-listeners)  
&nbsp;&nbsp;&nbsp;&nbsp;&bull; [services](#services)  
&nbsp;&nbsp;&nbsp;&nbsp;&bull; views  

## Particular patterns descriptions

### Concerns

Concerns are pieces of code which you can include to as many classes as you need, so you can stay [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself). It is also an elegant way you can avoid inheritance.

#### Code before concerns in use

    # app/models/article.rb:

    class Article < ActiveRecord::Base
      has_many :comments, as: :commentable

      def find_first_comment
        comments.first(created_at DESC)
      end

      def self.least_commented
        # too lazy to implement
      end
    end

#### Concerns in use

    # app/models/article.rb:

    class Article < ActiveRecord::Base
      include Commentable
    end


    # app/models/news.rb

    class News < ActiveRecord::Base
      include Commentable
    end


    # app/models/concerns/commentable.rb:

    module Commentable
      extend ActiveSupport::Concern

      included do
        has_many :comments, as: :commentable
      end

      def find_first_comment
        comments.first(created_at DESC)
      end

      module ClassMethods
        def least_commented
          # too lazy to implement
        end
      end
    end

### Form Objects

Sometimes you can have a form which is not possible to map to one entity. You can have form with two or three or more entities in one form and those entities doesn't even need to have a relation between each other. In such case you can use a form object and still use standard Rails validations.

    # app/models/register_form.rb:

    class RegisterForm
      include ActiveModel::Model

      validates_presence_of :name
      validates_presence_of :description

      def user
        @user ||= User.new
      end

      def note
        @note ||= Note.new
      end

      def submit(params)
        user.attributes = params.slice(:name)
        note.attributes = params.slice(:description)

        if valid?
          user.save!
          note.save!
          true
        else
          false
        end
      end
    end


    # app/views/users/new.html.erb:

    <%= form_for @register_form do |f| %>
      <%= f.text :name %>
      <%= f.text :description %>
      <%= f.submit "Register" %>
    <% end %>


    # app/controllers/users_controller.rb

    def new
      @register_form = RegisterForm.new
    end

    def create
      @register_form = RegisterForm.new
      if @register_form.submit(params[:user])
        session[:user_id] = @register_form.user.id
        redirect_to @register_form.user
      else
        render "new"
      end
    end

### Decorators

Decorators add another thin presentation layer in which you can adjust view output based on some decission-making, that exactly shouldn't be done in helpers. A typical example is showing different html based on same state.

#### Without Decorators

    # app/controllers/articles_controller.rb:

    def show
      @article = Article.find(params[:id])
    end


    # app/controllers/articles_helper.rb:

    def publication_status(article)
      if article.published?
        "Published at #{article.published_at.strftime('%A, %B %e')}"
      else
        "Unpublished"
      end
    end


    # app/views/articles/show.html.erb:

    <div class="time">
      <%= publication_status @article.published_at %>
    </div>

#### With using Draper

    # app/decorators/article_decorator.rb:

    class ArticleDecorator < Draper::Decorator
      delegate_all

      def publication_status
        if published?
          "Published at #{published_at}"
        else
          "Unpublished"
        end
      end

      def published_at
        object.published_at.strftime("%A, %B %e")
      end
    end


    # app/controllers/articles_controller.rb:

    def show
      @article = Article.find(params[:id]).decorate
    end


    # app/views/articles/show.html.erb:

    <div class="time">
      <%= @article.publication_status %>
    </div>

### Helpers

Helpers are methods accessible in views usually accepting some value and returning it back in some modified form.  
It should be used for adjusting the output, no business logic should be involved here.  Very often it is used for formatting.

    # app/helpers/application_helper.rb:

    class ApplicationHelper
      def humanize_date
        date.strftime("%d. %m. %Y")
      end
    end


    # app/views/articles/show.html.erb:

    <div class="time">
      <%= humanize_date @article.published_at %>
    </div>

### Policies

Policies solve authorizations. It is very close to decorators, but decorators are solving the view logic (what to show based on some conditions), policies are solving the authorization logic instead (if the user has right to see some piece of page or execute an action).  
The boundary between decorators and policies is very thin.

#### Authorization without policies

    # app/views/articles/show.html.erb:

    <div class="buttons">
      <% if not @article.published? && @user.admin? %>
        <%= button_to article_edit(@article), method: :get %>
      <% end %>
    </div>

#### Authorization using policies (without Pundit)

    # app/policies/article_policy.rb:

    class ArticlePolicy
      attr_reader :user, :article

      def initialize(user, article)
        @user = user
        @article = article
      end

      def update?
        user.admin? or not article.published?
      end
    end


    # app/controllers/articles_controller.rb:

    def show
      @article = Article.find params[:id]
      @article_policy = ArticlePolicy.new current_user, @article
    end


    # app/views/articles/show.html.erb:

    <div class="buttons">
      <% if not @article_policy.update? %>
        <%= button_to article_edit(@article), method: :get %>
      <% end %>
    </div>

#### Authorization using Pundit

    # app/policies/article.rb:

    class ArticlePolicy < ApplicationPolicy
      # automatically gives argument user
      # and article, based on file name
      def update?
        user.admin? or not article.published?
      end
    end


    # app/controllers/articles_controller.rb:

    def show
      @article = Article.find params[:id]
      authorize @article
    end


    # app/views/articles/show.html.erb:

    <div class="buttons">
      <% if policy(@article).update? %>
        <%= button_to article_edit(@article), method: :get %>
      <% end %>
    </div>

### Publishers and Listeners

Publishers and Listeners are two components of eventing functionality. Imagine an example when you delete an article. When the deletion is executed you want other system parts can react somehow (for example send an email).  
The important thing is, you don't want to hardcode those reactions to the deletion code. You just send a message (publisher) to anyone who is listening (listeners).

####  Simple Listeners

    # app/models/article.rb:

    class Article < ActiveRecord::Base
      after_commit :notify_editors,     on: :create
      after_commit :generate_feed_item, on: :create

      private
      def notify_editors
        EditorMailer.send_notification(self).deliver_later
      end

      def generate_feed_item
        FeedItem.create(self)
      end
    end

#### Listeners using Wisper

    # config/initializers/subscribers.rb:

    Article.subscribe(ArticleListener) / published synchronously
    # Article.subscribe(ArticleListener, async: true) / for asynchronous publishing


    # app/models/article.rb:

    class Article < ActiveRecord::Base
      include Wisper.model
    end


    # app/listeners/article_listener.rb:

    class ArticleListener
      def self.after_create(article)
        EditorMailer.send_notification(article).deliver_later
        FeedItem.create(article)
      end
    end

#### Decoupled publishers with Wisper

    # app/publishers/article.rb:

    class DeleteArticle
      include Wisper::Publisher

      def call(article_id)
        article = Article.find_by_id(article_id)

        # do some evil things with the article

        if article.deleted?
          broadcast(:delete_successful, article.id)
        else
          broadcast(:delete_failed, article.id)
        end
      end
    end


    # app/controllers/articles_controller.rb:

    def destroy
      delete_article = DeleteArticle.new

      delete_article.subscribe(ArticleMailer,      async: true)
      delete_article.subscribe(StatisticsRecorder, async: true)

      delete_article.on(:delete_successful) { |article_id| redirect_to order_path(order_id) }
      delete_article.on(:delete_failed)     { |article_id| render action: :new }

      delete_article.call(params[:id])
    end

### Services

If the logic in your controller is starting to blow up, it is time to extract it so a separate service object.

    # app/controllers/user_controller.rb:

    def show
      @user = GetUserAndNotify.find(params[:id])
    end


    # app/services/get_user_and_notify.rb:

    class GetUserAndNotify
      def self.find(id)
        user = User.find id
        if user.notify_on_get?
          Logger.log "User #{user.name} with id #{user.id} was requested"
          EventNotifier.log_event user.class.name, user.id
        end
        user.get_count += 1
        user.save
        user
      end
    end

> This sheet was created by Jiří Procházka https://www.linkedin.com/in/jiriprochazka/ and is based on the lecture of Honza Minárik https://www.linkedin.com/in/janminarik which he has made for CSRUG. It was created with his permission.
