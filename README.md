# Specification Pattern in C Sharp

Specification pattern, learn it, love it, and then probably overuse it till you sort out where it applies. The specification pattern is a fantastic pattern for taking a commonly used piece of business logic and wrapping it up into a reusable component.

What you’ll need are the specification classes included in this repository. Let’s take a look at a quick example:

```
public class UserCredentials
{
	public string Username { get; set; }
	public string Password { get; set; }
}
```

Maybe we’ve got some controller or class that does customer signups filling these user credentials with user input. However, we never trust users because they are all liars. All of them. Even your dear aunt Sally. She’s the worst.

Anyway, so you’ll probably end up having to do some checks to make sure the credentials have input and maybe abide by some standards for password length or whatever have you.

```
public class UserRegistration
{
	private readonly IUserService _userService;
	private readonly IBadWordsService _badWordsService;

	public UserRegistration(
		IUserService userService,
		IBadWordsService badWordsService)
	{
		_userService = userService;
		_badWordsService = badWordsService;
	}

	public bool Signup(UserCredentials userCredentials)
	{
		if(string.IsNullOrEmpty(userCredentials.Username)) 
			return false;

		if(string.IsNullOrEmpty(userCredentials.Password))
			return false;

		if(IsPasswordLongEnough(userCredentials.Password))
			return false;

		if(_badWordsService.HasBadWords(userCredentials.Username))
			return false

		if(!_userService.IsUsernameAvailable(username))
			return false;

		// finally ready to register
		return _userService.Register(username);
	}

	public bool IsPasswordLongEnough(string password)
	{
		if(string.IsNullOrEmpty(password))
			return false;

		return password.Length > 8;
	}
}
```

See the problem? The “UserRegistration” class needs to know about password validation, and whether a username is available or contains bad words. Likely means somewhere buried in some other developers code are the EXACT same checks. Worse they don’t do all the validation you’re supposed to do.

In comes the specification pattern to the rescue! Now we can take the common validation code and move it to a reusable class.

```
public class HasValidPassword : CompositeSpecification<UserCredential>
{
	public override bool IsSatisfiedBy(UserCredentials userCredentials)
	{
		if(string.IsNullOrEmpty(userCredentials.Password))
			return false;

		return IsPasswordLongEnough(userCredentials.Password);
	}

	private bool IsPasswordLongEnough(string password)
	{
		if(string.IsNullOrEmpty(password))
			return false;

		return password.Length > 8;
	}
}
```

```
public class HasAvailableUsername : CompositeSpecification<UserCredential>
{
	private readonly IUserService _userService;

	public HasValidUsername(IUserService userService)
	{
		_userService = userService;
	}

	public override bool IsSatisfiedBy(UserCredentials userCredentials)
	{
		if(string.IsNullOrEmpty(userCredentials.Username)) 
			return false;

		if(_badWordsService.HasBadWords(userCredentials.Username))
			return false

		return !_userService.IsUsernameAvailable(username));
	}
}
```

Thanks to the specification pattern the logic around password validity and username availability. Not to mention making sure no poo related usernames fill the database. 

Whelp, these classes are useless unless we actually use them. Let’s see what that looks like now in the UserRegistration class.

```
public class UserRegistration
{
	private readonly IUserService _userService;
	private readonly HasValidPassword _hasValidPassword;
	private readonly HasAvailableUsername _hasAvailableUsername;

	public UserRegistration(
		IUserService userService,
		HasValidPassword hasValidPassword,
		HasAvailableUsername hasAvailableUsername)
	{
		_userService = userService;
		_hasValidPassword = hasValidPassword;
		_hasAvailableUsername = hasAvailableUsername;
	}

	public bool Signup(UserCredentials userCredentials)
	{
		if(!_hasAvailableUsername
			.And(_hasValidPassword)
			.IsSatisfiedBy(userCredentials))
		{
			return false;
		}

		// finally ready to register
		return _userService.Register(username);
	}
}
```

Awesome right? 

The best part is now the specification classes can be shared in other areas of code. With the specification classes you can build some very complex business logic around the ability to chain together And / Or / and Not operations with independent pieces of code. Exactly like lincoln logs. But with code.
