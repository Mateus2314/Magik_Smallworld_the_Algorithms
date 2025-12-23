_pragma(classify_level=basic, topic={business_rule})
_method eo_meter.br_check_duplicate_meter_number()
    ## Validates if another meter with the same number already exists
    ## Field: meter_number must be unique

    _local meter_number << _self.meter_number

    _if meter_number _is _unset _orif meter_number = ""
    _then
        _return  # No number, skip duplicate validation
    _endif

    # Search for meters with same number
    _local predicate << predicate.eq(:meter_number, meter_number)
    _local duplicates << _self.source_collection.select(predicate)

    _for dup _over duplicates.fast_elements()
    _loop
        # Ignore itself (in case of update)
        _if dup = _self _then _continue _endif

        _local msg << write_string(
            "A meter with number '", meter_number, "' already exists. ",
            "Duplicate Asset ID: ", dup.asset_id)

        condition.raise(:user_error, :string, msg)
    _endloop

_endmethod
$

_pragma(classify_level=basic, topic={business_rule})
_method eo_service_point.br_check_duplicate_service_point()
    ## Validates if service point already exists at the same location
    ## Tolerance: 0.5 meters

    _local TOLERANCE << 0.5  # meters

    _if _self.location _is _unset
    _then
        _return
    _endif

    _local my_location << _self.location
    _local my_world << _self.source_view.world

    # Search for nearby points
    _local search_bounds << my_location.bounds.buffered_by(TOLERANCE)
    _local nearby_sp << my_world.geometry_at_bounds(
        search_bounds,
        eo_service_point.collection_name)

    _for other_sp _over nearby_sp.fast_elements()
    _loop
        _if other_sp = _self _then _continue _endif

        _local distance << my_location.distance_to(other_sp.location)

        _if distance < TOLERANCE
        _then
            _local msg << write_string(
                "A service point already exists very close by. ",
                "Distance: ", distance.write_string, " meters. ",
                "Asset ID: ", other_sp.asset_id)

            condition.raise(:user_error, :string, msg)
        _endif
    _endloop

_endmethod
$

_pragma(classify_level=basic, topic={business_rule})
_method eo_project.br_check_duplicate_project_name()
    ## Validates duplicate project name within the same status
    ## Allows same name in different statuses (e.g., Active vs Archived)

    _local project_name << _self.project_name
    _local project_status << _self.status

    _if project_name _is _unset _orif project_name = ""
    _then
        _return
    _endif

    _local pred << predicate.and(
        predicate.eq(:project_name, project_name),
        predicate.eq(:status, project_status))

    _local duplicates << _self.source_collection.select(pred)

    _for dup _over duplicates.fast_elements()
    _loop
        _if dup = _self _then _continue _endif

        _local msg << write_string(
            "A project with name '", project_name,
            "' already exists in status '", project_status, "'")

        condition.raise(:user_error, :string, msg)
    _endloop

_endmethod
$

## Installation script - Duplicate Validation
_block
    _local duplicate_rules << {
        # PRE INSERT
        {:br_meter_dup_insert, "eo_meter", "insert", "br_check_duplicate_meter_number()"},
        {:br_sp_dup_insert, "eo_service_point", "insert", "br_check_duplicate_service_point()"},
        {:br_proj_dup_insert, "eo_project", "insert", "br_check_duplicate_project_name()"},

        # PRE UPDATE
        {:br_meter_dup_update, "eo_meter", "update", "br_check_duplicate_meter_number()"},
        {:br_sp_dup_update, "eo_service_point", "update", "br_check_duplicate_service_point()"},
        {:br_proj_dup_update, "eo_project", "update", "br_check_duplicate_project_name()"}
    }

    _for rule_data _over duplicate_rules.fast_elements()
    _loop
        _local (rule_name, table_name, operation, method_name) << (_scatter rule_data)

        _local existing << sw_gis!business_rule.get_by_name(rule_name)
        _if existing _isnt _unset
        _then
            existing.delete()
        _endif

        sw_gis!business_rule.new_with(
            rule_name,
            table_name,
            "pre",
            operation,
            method_name,
            _true
        )

        write("✓ Duplicate check rule created: ", rule_name)
    _endloop

    gis_program_manager.databases[:gis].commit()
    write("All duplicate validation rules created!")

_endblock
$
