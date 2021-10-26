1. Check for Unauthorized Access
Always verify that the user is authorized to do the action he's doing. Say you have a page with a list of projects that a user owns, one link might be to /projects/1. However, the user can easily go to a different project page by changing 1 to any number.

Instead of @project = Project.find(params[:id]), you should use @project = @current_user.projects.find(params[:id]). If the project doesn't belong to the user, the latter code will return nil.

2. Authentication
For authentication, it is recommended you use an existing gem like devise or authlogic. If you create your own authentication system, use Rails' built-in has_secure_password. Devise and has_secure_password use bcrypt to hash the password. Never save passwords in clear text.

Save the hash of the password instead of the actual password. When authenticating a user, get the hash of the password and compare it with the hash on the database.

In the event an attacker gets access to your database, he'll need to compute the hash of possible passwords and compare it with the hash from your database. Bcrypt is designed to be computationally expensive so this can take a long time.

A salt is also recommended. This is a small random data that is combined with the password before it is passed to bcrypt.

3. Filter Passwords and Other Sensitive Data on Logs
Rails logs all requests to your application. When a user logs in, the username and password will be logged unless you filter the password. Rails by default creates config/initializers/filter_parameter_logging.rb which has

Rails.application.config.filter_parameters += [:password]
We're not saving the passwords in clear text on the database and we don't want them to appear on the logs either. You'll want to filter other sensitive data like credit cards.

4. Cross Site Request Forgery (CSRF)
Rails protects you from CSRF by adding an authenticity tokens on forms. You won't be able to submit to a POST action if you don't have the token. This is enforced by protect_from_forgery with: :exception that's added to application_controller.rb by default.

Do not skip this on your actions even for AJAX actions. When you use either rails-ujs or jquery_ujs, authenticity tokens will be added automatically.

5. Strong Parameters
A common pattern when creating or updating a record from a controller is

Person.create(params[:person])
This will raise an error. This uses mass assignment which may inadvertently update a value you don't wish your user to update. By using strong parameters, you whitelist the values that can be used.

params.require(:person).permit(:name, :age)
You're telling Rails that only name and age can be changed. If the user submits the form and for example, included admin with a value of true, you'll get an error. It doesn't matter that the form only has input for name and age. A user can easily change that to add more data.

6. Throttling Requests
On some pages like the login page, you'll want to throttle your users to a few requests per minute. This prevents bots from trying thousands of passwords quickly.

Rack Attack is a Rack middleware that provides throttling among other features.

Rack::Attack.throttle('logins/email', :limit => 6, :period => 60.seconds) do |req|
  req.params['email'] if req.path == '/login' && req.post?
end
7. Protecting Your Users
Your login page or forgotten password page should not give information if a user exists or not. Use a generic message if the username and password are wrong. If you use one message for the case when the username exists and the password is wrong, and a different message when the username doesn't exist, then an attacker can compile a list of usernames or emails of your users.

8. Use HTTPS
Use HTTPS for your whole site but at the very least for pages that deal with sensitive information like login or payment pages. If you use HTTP for a login page, anyone sniffing the network of your user will see the password in clear text.

The SSL certificate and key are not handled by Rails but on the webserver like nginx, or even above it on a load balancer (eg ELB).

On config/environments/production.rb, you can redirect all requests to HTTPS.

config.force_ssl = true
9. No Credentials in the Repository
The secret key base, database credentials, and other sensitive data should not be committed to your repository. By default, config/secrets.yml reads the secret key base from an environment variable in production. It is thus safe to commit secrets.yml to your repository.

If your database.yml contains your database credentials, it shouldn't be committed to your repository.

Rails 5.1 released a way to encrypt secrets. If you're using this feature you can commit the encrypted secrets file.

10. Credentials on Environment Variables
If you have a choice, do not put credentials on environment variables. If you need to use environment variables for your credentials, make sure that these environment variables are not leaked to 3rd party services, say for error reporting which usually logs all environment variables in the Rails app.

11. Rails Security Announcements
When a Rails security issue is identified and patched, you'll hear about it on this mailing list. If you're affected by a security issue, you should patch your Rails application or upgrade to the latest version immediately.

12. Bundler Audit
A new Rails 5.1 app depends on over 70 gems in development mode. It's not a stretch for a Rails app to depend on over 100 gems. To check if any of the gems you're using has a vulnerability issue, install the bundler-audit gem and run bundle audit.

Bundler-audit uses ruby-advisory-db, a database of security advisories related to Ruby libraries.

13. Run Brakeman
Brakeman is a static analysis security vulnerability scanner for Rails applications. While bundler-audit checks the Gemfile.lock only, brakeman checks your code.

I ran brakeman on one of my old Rails app, and it showed me possible issues with Cross Site Scripting, Denial of Service, Mass Assignment, and SQL Injection.

14. Admin Pages
Some apps provide an admin panel to superusers to manage data. Consider putting this on a different domain like admin.example.com. If the admin panel is separate from the main app, you can apply more restrictions like IP based filtering or require only connections through your VPN.

15. Run the Rails App as Unprivileged User
On your server, create a non-root user and use that to run your application. In case the app is compromised, the attacker won't be able to do as much damage as when he has root access.

16. SQL Injection
Some ActiveRecord methods don't escape arguments so you have to be careful when using them with user input. The ActiveRecord code below is common

e = params[:email]
User.where(email: e)
This is the safe version. Now take a look at the code below which you might think is equivalent.

e = params[:email]
User.where("email = '#{e}'")
where accepts a String that it passes to the WHERE clause of the SQL query. The 2 versions above look the same but if e is set to "') OR 1-- ", the query will become

SELECT `users`.* FROM `users` WHERE (email = '') OR 1-- ')
Anything after -- is a comment. OR 1 will select all the records from the users table.

Stick with the first version above which uses a hash. Another option is to use an array like

e = params[:email]
User.where("email = ?", e)
17. User Input
Rails by default escapes values on templates so this is not an issue anymore.

When you get input from your users like comments, sanitize the data when displaying them. They might enter html which can alter the design of the page or worse get data through Cross Site Scripting (XSS).
