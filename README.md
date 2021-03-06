# SQLiteORM

**SQLiteORM helps developers using SQLite in Android easier!** It's an ORM-like tool for multi table databases faster development.

## How to install
### in gradle
Add it in your root build.gradle at the end of repositories:
```
allprojects {
    repositories {
	...
	maven { url 'https://jitpack.io' }
    }
}
```
then add the depenceny
```
dependencies {
    compile 'com.github.mimrahe:sqliteorm:v1.0.5'
}
```
### in maven
```xml
<repositories>
    <repository>
	<id>jitpack.io</id>
	<url>https://jitpack.io</url>
    </repository>
</repositories>
```
then add the dependency
```xml
<dependency>
    <groupId>com.github.mimrahe</groupId>
    <artifactId>sqliteorm</artifactId>
    <version>v1.0.5</version>
</dependency>
```
### other ways
see [SQLiteORM on jitpack](https://jitpack.io/#mimrahe/sqliteorm)

## How to use
before using this library you must learn about using SQLite database in Android. see [Database in Android developers website](https://developer.android.com/training/basics/data-storage/databases.html)
### create version file
this ORM looks for sql files in assets folder of project. so create a new folder in assets and name it "database".
create a new file with extension ".sql" for each version of database.
for example for version 1 name file "1.sql".

### place sql statements in the file
for example if you want to create a new table in version 1 of database place this lines in it:
```sql
CREATE TABLE IF NOT EXISTS notes (_id INTEGER PRIMARY KEY, note VARCHAR(250) NOT NULL DEFAULT "", is_important BOOLEAN NOT NULL DEFAULT TRUE);
```
**Define sql file for each version of database**:

for example if your database version updated to 2 so create a sql file in "assets/database" and name it 2.sql

**Modify database via version files**:

for example if you want to create a new table and add new field to existed table do:
```sql
CREATE TABLE IF NOT EXISTS types (_id INTEGER PRIMARY KEY, type VARCHAR(50) NOT NULL);
ALTER TABLE notes ADD COLUMN type_id INTEGER DEFAULT 0;
```
### define a model for each table
models extend `ir.mimrahe.sqliteorm.ModelAbstract` abstract class:

**study this class so you can extend it and-or using that's methods!**
```java
package ir.mimrahe.sqliteorm;

import java.util.HashMap;

// this abstract class was defined in sqliteorm library
// so do not create this class
public abstract class ModelAbstract {
    /**
     * sets if and returns instance of model
     * @param id id of table
     * @return instance of model
     */
    public abstract ModelAbstract setId(Integer id);

    /**
     * @return fields will be inserted in table
     */
    public abstract HashMap<String, Object> getInsertFields();

    /**
     * @return fields will be updated
     */
    public abstract HashMap<String, Object> getUpdateFields();

    /**
     * @return table name of model
     */
    public abstract String getTableName();

    /**
     * @return primary key field name of table
     */
    public abstract String getPKName();

    /**
     * @return primaty key field value
     */
    public abstract String getPKValue();

    /**
     * checks if new value is dirty for update
     * @param newValue new value of field
     * @param oldValue old value of field
     */
    public boolean isDirty(Object newValue, Object oldValue){
        return oldValue != null && !newValue.equals(oldValue);
    }

    /**
     * @return copied instance of model
     */
    public abstract ModelAbstract copy();

    /**
     * @return instance of model
     */
    public abstract ModelAbstract getInstance();
    
    /**
     * searches table for primary key with specific value
     * @return found row Cursor object
     */
    public Cursor findWithPrimaryKey(){
        String whereClause = getPKName() + " = ?";
        String[] whereArgs = new String[]{getPKValue()};
        return DatabaseSingleton.getInstance().find(getTableName(), whereClause, whereArgs);
    }

    /**
     * save model in table
     * @return id of newly created row
     */
    public long save(){
        return DatabaseSingleton.getInstance().insert(getInstance());
    }

    /**
     * saves model in table and set id for model
     */
    public void saveAndSetId(){
        long id = save();
        setId((int) id);
    }

    /**
     * update model in table
     * @return the number of rows affected always equals 1
     */
    public int update(){
        String whereClause = getPKName() + " = ?";
        String[] whereArgs = new String[]{getPKValue()};
        return DatabaseSingleton.getInstance().update(getInstance(), whereClause, whereArgs);
    }

    /**
     * deletes model from table
     */
    public void delete(){
        String whereClause = getPKName() + " = ?";
        String[] whereArgs = new String[]{getPKValue()};
        DatabaseSingleton.getInstance().delete(getTableName(), whereClause, whereArgs);
    }
}
```
for example for table "notes":
```java
public class NoteModel extends ModelAbstract {
    private Integer id;
    private String note, dirtyNote;
    private Boolean isImportant, dirtyIsImportant;
    
    // use enum type for defining and using table column names!
    public enum Columns{
        ID("_id"),
        Note("note"),
	IsImportant("is_important");

        private String colName;

        Columns(String colName){
            this.colName = colName;
        }

        public String getColName(){
            return colName;
        }
    }

    NoteModel(){}

    NoteModel(String note, Boolean isImportant){
        this.note = note;
	this.isImportant = isImportant;
    }

    NoteModel(Integer id, String note, Boolean isImportant){
        this.id = id;
        this.note = note;
	this.isImportant = isImportant;
    }

    public Integer getId() {
        return id;
    }

    public String getNote() {
        return note;
    }

    public String getDirtyNote() {
        return dirtyNote;
    }
    
    public Boolean getIsImportant() {
        return isImportant;
    }

    public Boolean getDirtyIsImportant() {
        return dirtyIsImportant;
    }

    @Override
    public NoteModel setId(Integer id) {
        this.id = id;
        return this;
    }

    public NoteModel setNote(String note) {
        if (isDirty(note, this.note)){
            dirtyNote = note;
        }
        this.note = note;

        return this;
    }
    
    public NoteModel setIsImportant(Boolean isImportant) {
        if (isDirty(isImportant, this.isImportant)){
            dirtyIsImportant = isImportant;
        }
        this.isImportant = isImportant;

        return this;
    }
    
    public static NoteModel findWithId(Integer idValue){
        NoteModel noteModel = new NoteModel();
        noteModel.setId(idValue);
        Cursor found = noteModel.findWithPrimaryKey();

        try {
            if (found.moveToFirst()){
                noteModel.setNote(found.getString(found.getColumnIndex(Columns.Note.getColName())));
                noteModel.setIsImportant(found.getInt(found.getColumnIndex(Columns.IsImportant.getColName())) == 1);
            }
        } catch (Exception e){
            e.printStackTrace();
        } finally {
            if (found != null && !found.isClosed())
                found.close();
        }

        return noteModel;
    }

    public static ArrayList<NoteModel> findAll(){
        ArrayList<NoteModel> notes = new ArrayList<>();
        Cursor result = DatabaseSingleton.getInstance().findAll((new NoteModel()).getTableName());

        try {
            if(result.moveToFirst()){
                do {
                    Integer id = result.getInt(result.getColumnIndex(Columns.ID.getColName()));
                    String note = result.getString(result.getColumnIndex(Columns.Note.getColName()));
                    Integer importance = result.getInt(result.getColumnIndex(Columns.IsImportant.getColName()));
                    Log.e("in find all", importance.toString());
		    // boolean values in sqlite equals 0 for false and 1 for true
                    notes.add(new NoteModel(id, note, importance == 1));
                } while(result.moveToNext());
            }
        } catch (Exception e){
            Log.e("note model", "find all error");
        } finally {
            if (result != null && !result.isClosed()) {
                result.close();
            }
        }

        return notes;
    }

    @Override
    public HashMap<String, Object> getInsertFields() {
        HashMap<String, Object> insertFields = new HashMap<>();

        insertFields.put(Columns.Note.getColName(), getNote());
	insertFields.put(Columns.IsImportant.getColName(), getIsImportant());

        return insertFields;
    }

    @Override
    public HashMap<String, Object> getUpdateFields() {
        HashMap<String, Object> updateFields = new HashMap<>();
        // Note: use dirty values here!
        updateFields.put(Columns.Note.getColName(), getDirtyNote());
	updateFields.put(Columns.IsImportant.getColName(), getDirtyIsImportant());

        return updateFields;
    }

    @Override
    public String getTableName() {
        return "notes";
    }

    @Override
    public String getPKName() {
        return Columns.ID.getColName();
    }

    @Override
    public String getPKValue() {
        return getId().toString();
    }

    @Override
    public NoteModel copy() {
        return new NoteModel(getNote(), getIsImportant());
    }

    @Override
    public NoteModel getInstance() {
        return this;
    }

    @Override
    public String toString() {
        return "id: " + getId() + ", note: " + getNote() + ", my flag: " + getIsImportant();
    }
}
```
when you want to update a field of model new value gets in `dirty`. for example if you update `note` field new values gets in `note` and 
`dirtyNote`.

**define a `dirty` prefixed variable for fields that will be update.**

**`getUpdateFields` shoud place values of dirty fields in HashMap value places.**

**use `Integer` instead of `int`. use `Boolean` instead of `boolean`.**

### import sqliteorm in your class
```java
import ir.mimrahe.sqliteorm;
```

### init database
```java
String databaseName = "myDatabase";
int databaseVersion = 1;
DatabaseSingleton.init(getApplicationContext(), databaseName, databaseVersion);
```

### use model for CRUD operations
```java
NoteModel note1 = new NoteModel("call Ali today", true);
// note1.save(); or
note1.saveAndSetId(); // use this if you want to update

note1.setNote("call Ali today at 19:00");
note1.update();

note1.setNote("call Ali today at 19:00 and say hello").update();

NoteModel note2 = new NoteModel("go shopping", false);
note2.savAndSetId();

// creating new row in table and saving values of note2 instance in that
NoteModel note3 = note2.copy();
note3.save(); 

for(NoteModel note: NoteModel.findAll()){
    Log.e("all notes", note.toString());
}

// delete rows of table relates to note1 and note2 model instances
note1.delete();
// find note via ModelAbstract findWithPrimaryKey method
NoteModel noteInstance = NoteModel.findWithId(note2.getId());
Log.e("note 2 found", noteInstance.toString());
```

**when we need to edit model we need it's ID in the table; so we use `saveAndSetId` instead of `save`.**

### close database
close database and release resources
```java
    @Override
    protected void onDestroy() {
        DatabaseSingleton.closeDatabase();
        super.onDestroy();
    }
```

## Define new functions in models
use `DatabaseSingleton.getInstance()` for accessing database helper functions.
### functions in database helper
```java
// queries table
public Cursor find(
	String tableName, String selection, String[] selectionArgs,String groupBy, String having, String orderBy, String limit);
// queries table
public Cursor find(String tableName, String selection, String[] selectionArgs);
// queries all rows
public Cursor findAll(String tableName);
/** 
 * inserts into table
 * returns id of new row
 */
public long insert(ModelAbstract object);
/**
 * updates row
 * return 1
 */
public int update(ModelAbstract object, String whereClause, String[] whereArgs);
// deletes row
public void delete(String tableName, String whereClause, String[] whereArgs);
```
### call functions in this way
```java
DatabaseSingleton.getInstance().findAll();
```
### example of custom function in model
```java
    public static ArrayList<NoteModel> findAll(){
        ArrayList<NoteModel> notes = new ArrayList<>();
        Cursor result = DatabaseSingleton.getInstance().findAll((new NoteModel()).getTableName());

        try {
            if(result.moveToFirst()){
                do {
                    Integer id = result.getInt(result.getColumnIndex(Columns.ID.getColName()));
                    String note = result.getString(result.getColumnIndex(Columns.Note.getColName()));
                    notes.add(new NoteModel(id, note));
                } while(result.moveToNext());
            }
        } catch (Exception e){
            Log.e("note model", "find all error");
        } finally {
            if (result != null && !result.isClosed()) {
                result.close();
            }
        }

        return notes;
    }
```

## License
mimrahe/SQLiteORM is licensed under the
GNU General Public License v3.0

Permissions of this strong copyleft license are conditioned on making available complete source code of licensed works and modifications, which include larger works using a licensed work, under the same license. Copyright and license notices must be preserved. Contributors provide an express grant of patent rights.
