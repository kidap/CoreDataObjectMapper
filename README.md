# CoreDataObjectMapper
CoreDataObjectMapper


```
class CoreDataObjectMapper {

    static func serialize(_ object: NSManagedObject) -> [String: Any] {

        var dict: [String: Any] = [:]
        let entity = object.entity

        entity.attributesByName.forEach { (key, attribute) in

            switch attribute.attributeType {
            case .integer16AttributeType,
                 .integer32AttributeType,
                 .integer64AttributeType,
                 .decimalAttributeType,
                 .doubleAttributeType,
                 .floatAttributeType,
                 .stringAttributeType:
                dict[key] =  object.value(forKey: key)
            case .booleanAttributeType:
                dict[key] = NSNumber(value: object.value(forKey: key) as! Bool).intValue
            case .dateAttributeType:
                dict[key] = (object.value(forKey: key) as! Date).timeIntervalSince1970
            case .binaryDataAttributeType,
                 .transformableAttributeType,
                 .objectIDAttributeType,
                 .URIAttributeType,
                 .UUIDAttributeType: break
            default: break
            }
        }

        entity.relationshipsByName.forEach { (key, relationship) in
            if relationship.isToMany {
                let uuids = (object.value(forKey: key) as! Set<NSManagedObject>).map { $0.value(forKey: "uuid")! }
                dict[key] = uuids
            } else {
                let uuid = (object.value(forKey: key) as! NSManagedObject).value(forKey: "uuid")
                dict[key] = uuid
            }
        }

        return dict
    }

    static func deserialize<T: NSManagedObject>(_ dict: [String: Any], context: NSManagedObjectContext) -> T {

        let entity = T.entity()
        let object = NSManagedObject(entity: entity, insertInto: context)

        dict.keys.forEach { (key) in
            if let attribute = entity.attributesByName[key] {

                switch attribute.attributeType {
                case .integer16AttributeType,
                     .integer32AttributeType,
                     .integer64AttributeType,
                     .decimalAttributeType,
                     .doubleAttributeType,
                     .floatAttributeType,
                     .stringAttributeType:
                    object.setValue(dict[key], forKey: key)
                case .booleanAttributeType:
                    let boolValue = NSNumber(value: dict[key] as! Int).boolValue
                    object.setValue(boolValue, forKey: key)
                case .dateAttributeType:
                    let date = Date(timeIntervalSince1970: dict[key] as! TimeInterval)
                    object.setValue(date, forKey: key)
                case .binaryDataAttributeType,
                     .transformableAttributeType,
                     .objectIDAttributeType,
                     .URIAttributeType,
                     .UUIDAttributeType: break
                default: break
                }
            } else if let relationship = entity.relationshipsByName[key] {
                if relationship.isToMany {
                    let fetchRequest = NSFetchRequest<NSManagedObject>(entityName: relationship.destinationEntity!.name!)
                    fetchRequest.predicate = NSPredicate(format: "uuid IN %@", dict[key] as! [String])

                    if let relationshipObjects = try? context.fetch(fetchRequest){
                        object.setValue(Set(relationshipObjects), forKey: key)
                    }
                } else {
                    let fetchRequest = NSFetchRequest<NSManagedObject>(entityName: relationship.destinationEntity!.name!)
                    fetchRequest.predicate = NSPredicate(format: "uuid == %@", dict[key] as! String)

                    if let relationshipObjects = try? context.fetch(fetchRequest), let relationshipObject  = relationshipObjects.first {
                        object.setValue(relationshipObject, forKey: key)
                    }
                }
            }
        }

        return object as! T
    }
}
```
