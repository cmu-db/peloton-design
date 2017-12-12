# Zone Maps

## Overview

Zone Maps are implemented to improve scan times for predicates that might be highly selective. Zone Maps store metadata information about each column in the tile group like min and max. This can be used to skip a tile group based on the predicate value. Thus for highly selective predicate values we might ending up skipping a large number of tile groups improving our scan times tremendously.

## Scope
The implementation of Zone Maps in Peloton can be broken down into the following:-
1. **Zone Map Manager** : Any part of the system (ex Brain, Table Scan etc) should use the APIs provided by this Manager. This manager/indirection provides cleaner APIs, hiding the dirty work behind it. Using these APIs one can perform.
   - Creation of Zone Maps for a table / tile group . 
   - Deletion of a Zone Map for a Tile group.
   - Reading of Zone Map and comparing against Predicates
   - *Any other feature that goes in the zone map later should be added in the manager itself*
2. **Zone Map Catalog** :  Peloton stores the zone maps of the entire system in a single catalog table. Each row of this catalog has the following columns.
   - database_id 
   - table_id
   - tile_group_id
   - column_id
   - minimum
   - maximum
   - type
   - *In the absence of Array Types, we are currently storing minimum and maximum as VARCHAR and hence we store the type to deserialize it.*
3. **Additional Wiring / CallBacks / Proxies** : Primarily in place for the beast codegen to consume the Zone Maps. While iterating over the tilegroups, the decision to skip/scan a tile group is done through a simple call. This just adds one line of code in the `table.cpp` while iterating over the tile groups to scan. *Sidenote: If at some point its decided to not use Zone Maps, you can just delete this one line but this would break my heart*

## Architectural Design
The overall flow of Peloton is mostly unchanged except the following:-
1. In `table_scan_translator.cpp` we check if the predicate is Zone Mappable. If yes then we parse the predicates to get an array of predicates in which each element is a struct having the following parameters:
   - column_id
   - Comparison Operator ID
   - predicate value
2. In `table.cpp` we invoke the Zone Map Manager while iterating over the Tile Groups to return a True/False. A True indicates we need to scan the tile group and vice versa. The API(`ComparePredicateAgainstZoneMap`) which returns the True/False takes the predicate array and compares with the metadata present in the Zone Map Catalog.

## Design Rationale
The Zone Maps are stored in catalog to make them transactional. Earlier designs which stored Zone Maps on the heap with each tile group header having a pointer to the Zone Map was removed because it did not provide the transaction semantics. With Zone Maps in Catalog, we get the transactional capabilities for free.

## Testing Plan
1. Basic Zone Map Contents Test.
   - This checks whether the min and max for all tile groups and columns are accurate.
2. Transaction Level GC Manager Test - for Immutability
   - Creates a table and makes some tile groups immutable.
   - Deletion of slots in immutable tile groups are checked for no recycling.
3. Comparing different predicates against zone maps.
   - Single Column Predicates on (col_a = x, col_a < x, col_a > x)
   - Single Column Conjunction Predicate ( col_a > x and col_a < y)
   - Multi Column Conjunction Predicate  ( col_a > x and col_a < y and col_b > p and col_b < q)
4. End to End Tests with Zone Map
   - Conjunction and Simple Predicate tests just like table scan translator tests but now with Zone maps created which check for correctness.

## Trade-offs and Potential Problems
By removing the storage of zone maps in catalog we trade off some performance for transaction semantics. Some potential problems are also discussed in Future Work. Storage in catalog in the absence of array types also incurs an over head of serialization and deserialization of the min and max from VARCHAR to their original types.

## Future Work
1. Replacing the BWTree Index lookup with a Hash Map lookup to provide O(1) lookup and unnecessary copying of keys for index lookup.
2. Storing all column metadata information (min/max) in a single row in catalog once array types are implemented. This saves us multiple trips to the catalog for one tile group if we have predicates on multiple columns.
3. This should be checked and then done. If we can invoke the Zone Map manager before iterating over the tile groups and ask it to give the list of tile groups that need to be scanned it might give us better performance over invoking Zone Map manager's API each time while iterating over the tile groups. The benefit could be there if we store all tile groups and each columns metadata (min, max) in a single row. Thus in one trip to the catalog we could potentially get the metadata information about all tile groups. Then we could compare against the predicate and return the list of tile groups to scan.

