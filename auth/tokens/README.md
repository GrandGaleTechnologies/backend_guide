# How to Implement Access and Refresh Token Flow

**Title**: How to Implement Access and Refresh Token Flow  
**Created On**: 27-09-2024  
**Created By**: Bello Shehu Ango  
**Last Modified On**: 

### Modifiers:
- **Mod 1**:  
  **Name**:  
  **Date**:  
  **Summary of change**:  

## INTRODUCTION
This document discusses how to implement access and refresh tokens in GrandGale Backend systems. The main goal is to provide engineers with an overview of how access and refresh tokens are handled in the organization and to serve as a source of truth.

## GOALS
The main goals of the access and refresh token system are:
- To implement a secure and clean access-refresh token system that will provide the maximum amount of security.
- To make session management as easy as possible for frontend engineers.
- To allow us to force logouts across the entire system without hassle.

## IMPLEMENTATION

### Environment Variables:
- `SECRET_KEY`
- `ACCESS_TOKEN_EXPIRE_MIN`
- `REFRESH_TOKEN_EXPIRE_HOUR`

### Database Tables:
```sql
<object>_refresh_tokens: 
id INTEGER SERIAL PK
<object>_id INTEGER/UUID FOREIGN KEY NOT NULLABLE
token VARCHAR NOT NULLABLE
is_active BOOLEAN DEFAULT 'true' NOT NULLABLE
created_at DATETIME WITH TIMEZONE NOT NULLABLE
```
**Notes**:  
`<object>` refers to the user type, such as `user` or `admin`. For example, `user_refresh_tokens` or `admin_refresh_tokens`.

### Sample Access Token Body:
```json
{
  "type": "access",
  "ref_id": "{DB_ID_OF_REFRESH_TOKEN}",
  "sub": "{OBJECT_TYPE}-{OBJECT_ID}",
  "iat": "{UNIX_TIMESTAMP_OF_TIME_CREATED}",
  "exp": "{UNIX_TIMESTAMP_OF_EXPIRY_TIME}",
  "iss": "{URL_OF_APPLICATION}"
}
```

**Example**:
```json
{
  "type": "access",
  "ref_id": "24",
  "sub": "USER-45",
  "iat": "1725148800",
  "exp": "1727740799",
  "iss": "grandgale.com"
}
```

**Notes**:
- `type`: The JWT token type, either `access` or `refresh`.
- `ref_id`: The refresh token ID from the database.
- `sub`: The subject of the token, typically the user ID, prefixed with the object type (e.g., `ADMIN`, `STAFF`, `USER`).
- `iat`: Issued at timestamp (Unix).
- `exp`: Expiry timestamp (Unix).
- `iss`: Issuer of the token, typically the system URL.

### Sample Refresh Token Body:
```json
{
  "type": "refresh",
  "sub": "{OBJECT_TYPE}-{OBJECT_ID}",
  "iat": "{UNIX_TIMESTAMP_OF_TIME_CREATED}",
  "exp": "{UNIX_TIMESTAMP_OF_EXPIRY_TIME}",
  "iss": "{URL_OF_APPLICATION}"
}
```

**Example**:
```json
{
  "type": "refresh",
  "sub": "USER-45",
  "iat": "1725148800",
  "exp": "1727740799",
  "iss": "grandgale.com"
}
```

**Notes**: Refresh tokens do not include the `ref_id` key.

## Logic for Creating Access and Refresh Tokens:
1. Get user object.
2. Create a refresh token.
3. Save refresh token to the database.
4. (If implementing a max number of signed-in devices) Check if the user has exceeded the maximum number of refresh tokens.
   - If so, invalidate the old refresh token when generating a new one.
5. Create an access token.
6. Set the `ref_id` to the refresh token’s primary key.
7. Return both tokens.

**Example Code (Python)**:
```python
# Get user
user = await selectors.get_user_by_email(email=email, db=db)

# Situation 1 -> Check if user has maxed out refresh tokens
if db.query(models.UserRefreshToken).filter_by(user_id=user.id).count() > MAX_COUNT:
    raise MaxSignInExceeded()

# Situation 2 -> Invalidate old refresh tokens
db.query(models.UserRefreshToken).filter_by(user_id=user.id).update({"is_active": False})
db.commit()

# Create refresh token
ref_token = await create_token(type="refresh", sub=f"USER_{user.id}")

# Create refresh token object
ref_token_obj = await create_user_refresh_token(user=user, token=ref_token, db=db)

# Create access token
access_token = await create_token(type="access", sub=f"USER-{user.id}", ref_id=ref_token_obj.id)
```

## Logic for Validating Access and Refresh Tokens:

### For Access Tokens:
1. Decode the token.
2. Get the token’s `ref_id`.
3. Retrieve the refresh token object using `ref_id`.
4. Check if the token is expired or inactive.
   - If inactive, raise a 401 error.
   - If expired, raise a 401 error.
5. Get the token's `sub` and split it into the object type and ID.
6. Ensure the `sub` is valid for the correct endpoint.
7. Retrieve the user object using the ID.
8. If the user doesn’t exist or is deactivated, raise a 401 error.
9. Return the user object.

**Example Code (Python)**:
```python
# Decode token
token = await decode_token(token=token)

# Get token ref_id
ref_id = token.get("ref_id")
if not ref_id:
    raise Unauthorized()

# Check: valid ref_id
ref_token_obj = await selectors.get_user_refresh_token_by_id(id=ref_id, db=db)

# Check: expired or inactive
if not ref_token_obj.is_active or ref_token_obj.created_at + timedelta(hours=REFRESH_TOKEN_EXPIRE_HOURS) > datetime.now():
    ref_token_obj.is_active = False
    db.commit()
    raise Unauthorized()

# Check: valid sub
sub = token.get("sub")
if not sub:
    raise Unauthorized()

# Check: valid sub head
sub_head, sub_id = token.split('-')
if sub_head != "USER":
    raise Unauthorized()

# Check: valid sub_id
user = await selectors.get_user_by_id(id=sub_id, db=db, raise_exc=False)
if not user:
    raise Unauthorized()

return user
```

### For Refresh Tokens:
1. Retrieve the refresh token object.
2. Check if it is expired or inactive.
3. Issue a new access token.

**Example Code (Python)**:
```python
# Get ref_token object
ref_token_obj = await selectors.get_user_refresh_token(token=token, db=db)

# Check: expired or inactive
if not ref_token_obj.is_active or ref_token_obj.created_at + timedelta(hours=REFRESH_TOKEN_EXPIRE_HOURS) > datetime.now():
    ref_token_obj.is_active = False
    db.commit()
    raise Unauthorized()

# Issue new access token
access_token = await create_token(type="access", sub=f"USER-{ref_token_obj.user_id}")

return access_token
```
