# Authentication

## Custom `AUTH_USER_MODEL` from the very first commit

Non-negotiable, even if it starts out identical to Django's default `User`.
Swapping `AUTH_USER_MODEL` after migrations have been applied in production
is a destructive migration — every foreign key to `User` in the database
needs to be recreated.

## `AbstractBaseUser` + `PermissionsMixin`, not `AbstractUser`

`AbstractUser` inherits fields that may not fit (a generic `username` when
login is by email, `first_name`/`last_name` fragmented when the domain
thinks in full names). Prefer `AbstractBaseUser` + `PermissionsMixin`:
you get the password system (`set_password`/`check_password`) and
permissions (`PermissionsMixin`) for free, and define every field from
scratch.

```python
# apps/accounts/models.py
class User(AbstractBaseUser, PermissionsMixin):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    email = models.EmailField(unique=True)
    full_name = models.CharField(max_length=255)
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)

    USERNAME_FIELD = "email"
    REQUIRED_FIELDS = ["full_name"]

    objects = UserManager()  # you must write create_user/create_superuser
```

```python
# settings
AUTH_USER_MODEL = "accounts.User"
```

## What to leverage from Django without reinventing it

- **Password hashing** (`set_password`/`check_password`, configurable
  `PASSWORD_HASHERS`) — don't reinvent this
- **The permission system** (`PermissionsMixin` gives you `is_superuser`,
  groups, `user.has_perm()`) — useful in the Admin too
- **`authenticate()`** and `AUTHENTICATION_BACKENDS` — pluggable (email +
  password today, SSO tomorrow) without rewriting the flow

## What we don't use, given stateless JWT

- `django.contrib.sessions` — not even in `MIDDLEWARE`, if the API is 100%
  JWT
- `django.contrib.auth.views` — prebuilt HTML views, don't make sense for
  a pure API

## Authentication is business logic — it lives in `services/`

Login, registration, password changes aren't an exception to the service
rule. Never business logic directly inside a `simplejwt` view.

```python
# apps/accounts/services/register.py
def register_user(email: str, password: str, full_name: str) -> User:
    if User.objects.filter(email=email).exists():
        raise UserAlreadyExistsError(email)

    with transaction.atomic():
        user = User.objects.create_user(email=email, password=password, full_name=full_name)

    transaction.on_commit(lambda: send_welcome_email.delay(user.id))
    return user
```

```python
# apps/accounts/services/authenticate.py
def authenticate_user(email: str, password: str) -> tuple[User, str, str]:
    user = authenticate(email=email, password=password)  # leverages the native backend
    if user is None:
        raise InvalidCredentialsError()
    if not user.is_active:
        raise UserInactiveError(user.id)

    refresh = RefreshToken.for_user(user)
    return user, str(refresh.access_token), str(refresh)
```

The service uses the native `authenticate()` internally, but the caller
doesn't need to know that. Swapping the authentication backend later (SSO,
LDAP) only changes this service — the view and the rest of the system stay
the same.

## Customizing `simplejwt` without fighting the library

Use the official extension point instead of rewriting token generation:

```python
# apps/accounts/tokens.py
class CustomTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        token["full_name"] = user.full_name
        token["is_staff"] = user.is_staff
        return token
```

## Refresh token in an httpOnly cookie

```python
response = Response({"access_token": access_token, "user": UserSerializer(user).data})
response.set_cookie(
    "refresh_token", refresh_token,
    httponly=True, secure=True, samesite="Strict",
    max_age=60 * 60 * 24 * 7,
)
```

The refresh view reads the cookie (not the request body), validates it via
`simplejwt`, and issues a new access token.

## Summary

| Rule | Reason |
|---|---|
| Custom `AUTH_USER_MODEL` from the start | Swapping it later is a destructive migration |
| `AbstractBaseUser` + `PermissionsMixin` | Full control over fields |
| Login/registration/password change are `services/` | Architectural consistency, testable outside HTTP |
| JWT customization via `simplejwt`'s `get_token()` | Leverages already-tested signing/expiration logic |
| Refresh token in an `httponly` + `secure` + `samesite=Strict` cookie | Reduces XSS exposure for the long-lived credential |
