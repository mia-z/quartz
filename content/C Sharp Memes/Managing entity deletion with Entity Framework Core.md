---
title: Soft deleting entities in C# and Entity Framework Core
draft: false
tags: 
  - CSharp
  - Entity Framework Core
  - SQL
  - ORM
--- 
## To delete or not delete?
### Deleting data (but not really)
Sometimes you probably don't want to actually delete data, but rather hide it from showing up and giving the illusion that it's deleted. A common way of doing this is simply adding a flag, or property, to an object denoting that it is deleted, or no longer active. 

```c#
public class User
{
	public string Username { get; set; }
	public string PasswordHash { get; set; }
	public bool Deleted { get; set; }
}
```

Here is a simple class object representing a user, with the relevant flag denoting whether or not the account is deleted. 
If you wanted, for example, to fetch all of the users but not include the deleted users you would simply tell the query to filter out where the `Deleted` property is set to `false`.

```sql
SELECT * FROM User
WHERE [Deleted] = false;
```

### In practice
These two additions might add some tedious overhead and/or boilerplate to your existing code, however we can leverage modern C# language concepts and Entity Framework Core to optimize it as much as possible.

### Identifying entities that can be deleted and (not)deleted
Rather than checking if the entity has a specific property called `Deleted` we can simply make the class implement from an `ISoftDelete` interface which defined a contract with a `IsDeleted` property and `DeletedAt` (as to keep more fine-grained track of deletion). 

```c#
public interface ISoftDelete
{
	public bool IsDeleted { get; set; }
	public DateTime? DeletedAt { get; set; } = null;
}
```

This would then make our new class like

```c#
public class User : ISoftDeleted
{
	public string Username { get; set; }
	public string PasswordHash { get; set; }
	
	public bool IsDeleted { get; set; }
	public DateTime? DeletedAt { get; set; } = null;
}
```

Although this _will_ add some extra boilerplate to your code, most modern IDE's will be able to auto-implement the interface members.

Now we no longer care about the entity itself, but rather if the entity is `ISoftDelete`.

```c#
if (entityToDelete is ISoftDelete entity)
{
	entity.IsDeleted = true;
	entity.DeletedAt = DateTime.Now;
}
```

### Saving (Deleting) entities
Normally with deleting entities in Entity Framework Core you notify the change tracker that the entity is deleted - and if we do that in our case, the entity we _dont_ want to delete will be deleted, so we would have to write specific code in each case to handle this. However we can use EF Core's interceptors to handle this for us, avoiding the need to write that specific code.
Normally we would write;

```c#
var userToDelete = await _users.GetUser(id);
userToDelete.State = EntityState.Deleted;
await _context.SaveChangesAsync();
```

But our ISoftDelete entities require a different approach to be able to soft delete them, and rather than writing this everywhere;

```c#
var userToDelete = await _users.GetUser(id);
userToDelete.Entity.IsDeleted = true;
userToDelete.Entity.DeletedAt = DateTime.Now;
userToDelete.State = EntityState.Modified;
await _context.SaveChangesAsync();
```

We can use EF Core's `SaveChangesInterceptor` which will run as a side effect to the `SaveChanges(Async)`Method

```c#
public class SoftDeleteInterceptor : SaveChangesInterceptor
{
	public override InterceptionResult<int> Savingchanges(DbContextEventData eventData, InterceptionResult<int> result)
	{
	   if (eventData.Contect is null)
	        return result;
	
	    foreach (var entry in eventData.Context.ChangeTracker.Entries())
	    {
	        if (entry is not { State: EntityState.Deleted, Entity: ISoftDelete entity })
	        	continue;
	
	        entry.State = EntityState.Modified;
	        userToDelete.Entity.IsDeleted = true;
	        userToDelete.Entity.DeletedAt = DateTime.Now;
	    }
			
	    return result;
    }
}
```

Which we can then register in the `OnConfiguring` method in our main DbContext file.

```c# title="ProjectDbContext.cs"
// Code omitted

protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
	//Code omitted
	
	optionsBuilder
	    .AddInterceptors(new SoftDeleteInterceptor());
}
```

So now whenever we tell the change tracker that an entity should be deleted and then save changes, it will check if the entity implement ISoftDelete and if it does then the entity will only be soft deleted as we specified!

### Bonus: adding default query filters
Rather than specifying everytime we want to fetch an `ISoftDelete`'able entity, if it _is_ in fact deleted, we can set the default query behaviour in the model creator.

```c# title="ProjectDbContext.cs"
// Code omitted

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
	//Code omitted
	
	modelBuilder.Entity<User>()
	    .HasQueryFilter(x => x.IsDeleted == false);
}
```

This will automatically filter out the users that have `IsDeleted` set to `true` so that we don't have to write that in the query every time -- especially in cases where it might be forgotten!