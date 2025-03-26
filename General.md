### Where is access token
An access token is stored in HTTP request's **Authorization header** (most secure). Also possible to store tokens in **cookies**, **query parameters** (less secure), **request body** (for some API calls, especially for OAuth 2.0 token exchanges) ![[Pasted image 20250317105359.png]]
### Real DB for unit testing
Normally not required since unit tests verify behavior of individual components in isolation. However, an in-memory db might behave differently than production db. Set up a real db in cases when testing:
1. **Database-Dependent Logic** - complex queries must match the actual db schema, db index and constraints might affect the results, in-memory db may behave differently than production db
2. **Model Persistence** - ensures that entity is actually persisted, check if db constraints (e.g., unique indexes) work correctly,  helps catch Hibernate (or different ORM) dialect issues early
3. **Transactions/Rollbacks** - ensures that rollback works as expected, simulates real-world transactional behavior
4. **Migration or Schema Changes** - check that migrations run correctly, data integrity is maintained
5. **Performance on Large Datasets** - ensures queries scale with production-sized data, allows index optimizations

In contrast, do **NOT** use real db if:
- you are running pure unit tests (mock the db/dao calls instead)
- tests run slowly
- tests modify shared data (avoid side effects in a shared environment)
- you are running tests in CI/CD pipelines unless properly manages

### No. of lines in a git repository/directory
Handy script that prints:
- total number of lines from each **first-level** sub-directory
- break-down of **individual files** with corresponding line count in each first-level sub-directory
- **grand total** of lines of code
```sh
find . -name "*.java" -print0 | xargs -0 wc -l | grep -v "total" | awk '
{
    if (NF > 1) {
        sub(/^\.\//, "", $2);  # Remove leading "./"
        split($2, parts, "/");
        first_level = parts[1];

        # Store file-specific line count
        file_list[first_level] = file_list[first_level] "\n    " $1 " " $2;
        
        # Sum lines per first-level directory
        dir_count[first_level] += $1;
        total += $1;
    }
}
END {
    for (dir in dir_count) {
        print dir_count[dir], dir;  # Print subtotal per directory
        print file_list[dir];       # Print individual file breakdown
    }
    print "Grand Total:", total;  # Print grand total
}'
```

The option `-type f` instead of `-name "*.java"` for command `find` will scan all regular files