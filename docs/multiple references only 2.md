# Multiple reference selection feature

having these tables:

rol
------
id  name

1 admin
2 user
3 viewer

user
------
id  name

1 harry
2 paco


user__rol
----------
    
id  id_user  id_role    [EntityAttribute]
1      1      1
2      2      1
3      1      2
4      2      2
 [x] --> marked as multiple
Fk: id_user -> user.id     [EntityAttributeReference]
Fk: id_role -> rol.id      [EntityAttributeReference]

when the user goes to the user__rol form
and enters
role: admin 
users: harry, paco   (can only enter multiple if the reference is multiple)

I want to insert the following rows:


user__rol
----------
admin harry
admin paco

-------------------