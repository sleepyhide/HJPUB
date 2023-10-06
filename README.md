# HJPUB
public
extends CharacterBody2D

class_name Player


const WALK_SPEED = 100
const SPRINT_SPEED = 170
const GROUND_TILE_ID = 1
const SOW_TILE_ID = 7
const WATERED_SOW_TILE_ID = 9
const MAX_WATER_LEVEL: int = 100
const WATER_TILE_ID = 8

var seeded_tiles: Dictionary = {}
var watering_can_water_level: int = 100
var player_velocity: Vector2 = Vector2.ZERO
var anim_player: AnimationPlayer
var weapon_anim_player: AnimationPlayer
var weapon: Sprite2D
var last_horizontal_direction: String = ""
var last_vertical_direction: String = ""
var is_sprinting: bool = false
var active_slot_index: int = -1
var state: String = "Idle"
var is_manual_animation_playing: bool = false
var current_manual_animation: String = ""
var idle_timer: Timer
var player_state: String = "IDLE"
var charge_level: int
var MAX_CHARGE_LEVEL: int = 100
var last_movement_direction: String = ""
@onready var skillbar = get_node("/root/World/HUD/Skillbar")
@export var inventory: Inventory
@onready var tile_map = $"../TileMap"
@onready var highlightTileSprite = $highlightTile/Sprite2D
@onready var seedTest = $seedTest
@onready var inventory_gui = get_node("$../HUD/InventoryGui")
@onready var slot_gui = $"../HUD/Slot"


func _ready():
	
	$RefillTimer.connect("timeout", Callable(self, "_on_RefillTimer_timeout"))
	highlightTileSprite.visible = false
	call_deferred("deferred_connect")
	$woodAxe.set_monitoring(false)
	$woodAxe.connect("area_entered", Callable(self, "_on_woodAxe_area_entered"))
	idle_timer = $IdleTimer
	idle_timer.connect("timeout", Callable(self, "_on_IdleTimer_timeout"))
	anim_player = get_node("AnimationPlayer")
	weapon = get_node("Weapon")
	weapon_anim_player = weapon.get_node("AnimationPlayer") 
	anim_player.connect("animation_finished", Callable(self, "_on_AnimationPlayer_animation_finished"))

func _on_update_progress(value):
	slot_gui.value = value
func deferred_connect():
	slot_gui.connect("update_progress", Callable(self, "_on_update_progress"))
	
func _physics_process(delta: float) -> void:
	if Input.is_action_pressed("ui_shift") and charge_level < MAX_CHARGE_LEVEL:
		charge_level += 10
		
		update_Playerprogress_bar()
		
	#print("players global pos:", global_position)
	#print(typeof(tile_map), tile_map is TileMap)
	if highlightTileSprite.visible:
		position_highlight_tile()
	
	if Input.is_action_just_pressed("ui_shift"):
		
		perform_item_action()
		
		
	if Input.is_action_just_pressed("ui_select"):
		is_sprinting
	
	if is_manual_animation_playing:
		return
	
	var anim_name: String = ""
	var weapon_anim_name: String = ""
	var current_speed: int = SPRINT_SPEED if is_sprinting else WALK_SPEED
	

	if Input.is_action_just_pressed("ui_select"):
		is_sprinting = not is_sprinting
		return

	if Input.is_action_just_pressed("slot_1"):
		select_slot(0)
	elif Input.is_action_just_pressed("slot_2"):
		select_slot(1)
	elif Input.is_action_just_pressed("slot_3"):
		select_slot(2)
	elif Input.is_action_just_pressed("slot_4"):
		select_slot(3)
	elif Input.is_action_just_pressed("slot_5"):
		select_slot(4)
		
	elif Input.is_action_just_pressed("ui_activate"):
		activate_slot()



	if Input.is_action_pressed("ui_left"):
		player_velocity.x = -current_speed
		last_movement_direction = "left"
		anim_name = "leftSprintAnim" if is_sprinting else "leftAnim"
		weapon_anim_name = "leftAnim"
		last_horizontal_direction = "left"
	elif Input.is_action_pressed("ui_right"):
		player_velocity.x = current_speed
		last_movement_direction = "right"
		anim_name = "rightSprintAnim" if is_sprinting else "rightAnim"
		weapon_anim_name = "rightAnim"
		last_horizontal_direction = "right"
	else:
		player_velocity.x = 0

	if Input.is_action_pressed("ui_up"):
		player_velocity.y = -current_speed
		last_movement_direction = "up"
		anim_name = "upSprintAnim" if is_sprinting else "upAnim"
		weapon_anim_name = "upAnim"
		last_vertical_direction = "up"
	elif Input.is_action_pressed("ui_down"):
		player_velocity.y = current_speed
		last_movement_direction = "down"
		anim_name = "downSprintAnim" if is_sprinting else "downAnim"
		weapon_anim_name = "downAnim"
		last_vertical_direction = "down"
	else:
		player_velocity.y = 0

	velocity = player_velocity
	move_and_slide()

	# Ccheck if player is idle
	if player_velocity == Vector2.ZERO and not is_manual_animation_playing:
		player_state = "IDLE"
	elif player_velocity != Vector2.ZERO:
		player_state = "MOVING"
	elif is_manual_animation_playing:
		player_state = "MANUAL_ANIM"
		
	match player_state:
		"IDLE":
			if last_vertical_direction == "down":
				anim_player.play("downIdleAnim")
			elif last_vertical_direction == "up":
				anim_player.play("upIdleAnim")
				weapon_anim_player.play("upIdleAnim")
			elif last_horizontal_direction == "left":
				anim_player.play("leftIdleAnim")
			elif last_horizontal_direction == "right":
				anim_player.play("rightIdleAnim")
		"MOVING":
			if anim_name != "" and anim_player.current_animation != anim_name:
				anim_player.play(anim_name)
				weapon_anim_player.play(weapon_anim_name)
				
		"MANUAL_ANIM":
			if anim_player.current_animation != current_manual_animation:
				anim_player.play(current_manual_animation)
			
	detect_current_tile()
			
			
			
			
func _input(event):
	if Input.is_action_just_released("ui_shift"):
		water_tiles(charge_level / 10)
		charge_level = 0
		update_Playerprogress_bar()

func is_shovel_equipped() -> bool:
	if active_slot_index != -1:
		var active_slot = skillbar.get_node("GridContainer").get_child(active_slot_index)
		if active_slot and active_slot.associated_slot and active_slot.associated_slot.item:
			return active_slot.associated_slot.item.name == "shovel"
	return false

func deduct_seed_from_active_slot():
	if active_slot_index != -1:
		var active_slot_gui = skillbar.get_node("GridContainer").get_child(active_slot_index)
		if active_slot_gui and active_slot_gui.associated_slot and active_slot_gui.associated_slot.item:
			active_slot_gui.associated_slot.amount -= 1
			if active_slot_gui.associated_slot.amount <= 0:
				active_slot_gui.associated_slot.item = null
				active_slot_gui.associated_slot.amount = 0
				active_slot_gui.update_slot(active_slot_gui.associated_slot)



func place_item():
	if active_slot_index != -1:
		var active_slot = skillbar.get_node("GridContainer").get_child(active_slot_index)
		if active_slot and active_slot.associated_slot and active_slot.associated_slot.item:
			var item = active_slot.associated_slot.item
			if item.name == "seedTest":
				var tile_coords = tile_map.local_to_map(tile_map.to_local(global_position))
				if tile_map.get_cell(tile_coords.x, tile_coords.y) == WATERED_SOW_TILE_ID:
					spawn_seed_on_tile(tile_coords)
					active_slot.associated_slot.deduct()
					active_slot.update_slot(active_slot.associated_slot)
					inventory_gui.update()



func detect_current_tile():
	var tile_coords = tile_map.local_to_map(tile_map.to_local(global_position))
	var tile_source_id = tile_map.get_cell_source_id(0, tile_coords)
	#print("Tile Source ID: ", tile_source_id)
	if (tile_source_id == GROUND_TILE_ID or tile_source_id == SOW_TILE_ID or tile_source_id == WATERED_SOW_TILE_ID):
		highlightTileSprite.visible = true
	else:
			highlightTileSprite.visible = false
		


	#print("players global pos:", global_position)
	


	var tile_data = tile_map.get_cell_tile_data(0, tile_coords)
	#print("Player is on the ground tile, global position")
	#print("tile coordinates:", tile_coords)
	

	

func  _on_hurt_box_area_entered(area):
	if area.has_method("collect"):
		area.collect(inventory)
		
		
func _on_IdleTimer_timeout():
	if not is_manual_animation_playing:
		if last_vertical_direction == "down" and last_horizontal_direction != "":
			anim_player.play(last_horizontal_direction + "IdleAnim")
		elif last_vertical_direction == "up":
			anim_player.play("upIdleAnim")
		elif last_horizontal_direction != "":
			anim_player.play(last_horizontal_direction + "IdleAnim")

func select_slot(index):
	if active_slot_index != -1 and active_slot_index < skillbar.get_node("GridContainer").get_child_count():
		var prev_slot = skillbar.get_node("GridContainer").get_child(active_slot_index)
		prev_slot.deselect()
		if prev_slot.associated_slot:
			prev_slot.associated_slot.selected = false
		
	if index < skillbar.get_node("GridContainer").get_child_count():
		var current_slot = skillbar.get_node("GridContainer").get_child(index)
		current_slot.select()
		if current_slot.associated_slot:
			current_slot.associated_slot.selected = true
		active_slot_index = index
	else:
		active_slot_index = -1


func spawn_seed_on_tile(tile_coords: Vector2):
	if seedTest and not seedTest.is_queued_for_deletion() and not seeded_tiles.has(tile_coords):
		var seed_instance = seedTest.duplicate()
		var local_position = tile_map.map_to_local(tile_coords)
		seed_instance.global_position = highlightTileSprite.global_position
		seed_instance.is_placed = true
		get_parent().add_child(seed_instance)
		seeded_tiles[tile_coords] = true
		deduct_seed_from_active_slot()
	
func gather_placed_seed(seed_instance):
	if seed_instance and not seed_instance.is_queued_for_deletion():
		if seed_instance.is_placed:
			seed_instance.is_being_used = false
			seed_instance.is_placed = false
			inventory.insert(seed_instance.itemRes)
			seed_instance.queue_free()
			
func refill_watering_can():
	var active_slot = skillbar.get_node("GridContainer").get_child(active_slot_index)
	if active_slot and active_slot.associated_slot and active_slot.associated_slot.item:
		var item = active_slot.associated_slot.item
		if item.name == "wateringCan" and item.durability < MAX_WATER_LEVEL:
			$RefillTimer.start()
			

func _on_RefillTimer_timeout():
	var active_slot = skillbar.get_node("GridContainer").get_child(active_slot_index)
	if active_slot and active_slot.associated_slot and active_slot.associated_slot.item:
		var item = active_slot.associated_slot.item
		if item.name == "wateringCan":
			item.durability += 25
			watering_can_water_level += 25
			update_progress_bar(item.durability)
			if item.durability >= MAX_WATER_LEVEL:
				$RefillTimer.stop()
				
func update_Playerprogress_bar():
	$TextureProgressBar.value = charge_level


func update_progress_bar(value: int):
	slot_gui.update_progress_bar(value)
	
func perform_item_action():
	if active_slot_index != -1:
		var active_slot = skillbar.get_node("GridContainer").get_child(active_slot_index)
		if active_slot and active_slot.associated_slot and active_slot.associated_slot.item:
			var item = active_slot.associated_slot.item
			var current_direction = determine_current_direction()
			if item.animations.has(current_direction):
				var anim_name = item.animations[current_direction]
				if anim_player.has_animation(anim_name):
					if item.name == "shovel":
						var tile_coords = tile_map.local_to_map(tile_map.to_local(global_position))
						if tile_map.get_cell_source_id(0, tile_coords) == GROUND_TILE_ID or SOW_TILE_ID:
							highlightTileSprite.visible = true
							position_highlight_tile()
							
							
						get_node("/root/World/Player/shovel")
						#print("Item name:", item.name)
						#print(highlightTileSprite.global_position)
					elif item.name == "woodAxe":
						$woodAxe.set_monitoring(true)
					elif item.name == "wateringCan":
						var tile_coords = tile_map.local_to_map(highlightTileSprite.global_position - tile_map.global_position)
						if tile_map.get_cell_source_id(0, tile_coords) == WATER_TILE_ID and Input.is_action_just_pressed("ui_shift"):
							print("Refilling watering can")
							refill_watering_can()
							active_slot = skillbar.get_node("GridContainer").get_child(active_slot_index)
						else:
							water_tile()
					
						
					elif item.name == "seedTest":
						print("SeedTest item detected")
						var tile_coords = tile_map.local_to_map(highlightTileSprite.global_position - tile_map.global_position)
						if not seeded_tiles.has(tile_coords):
							print("Trying to spawn seed at:", tile_coords)
							if tile_map.get_cell_source_id(0, tile_coords) == WATERED_SOW_TILE_ID:
								print("Correct tile found for seed spawning!")
								spawn_seed_on_tile(tile_coords)
								seedTest.is_being_used = true
								seedTest.is_placed = true
						
							else:
								print("Incorrect tile for seed spawning. Tile ID:", tile_map.get_cell_source_id(0, tile_coords))
						else:
							
							highlightTileSprite.visible = false
					
					is_manual_animation_playing = true
					current_manual_animation = anim_name
					print("playing animation:", anim_name)
					anim_player.play(anim_name)
					idle_timer.start()
					
				else:
					print("animation not found:", anim_name)
			else:
				print("direction not found in item animations", current_direction)
		else:
			print("no item associated with the active slot")
	else:
		print("no active slot selected")
		

	
func determine_current_direction():
	if last_vertical_direction != "":
		return last_vertical_direction
	elif last_horizontal_direction != "":
		return last_horizontal_direction
	else:
		return "down"
	
func activate_slot():
	if active_slot_index != -1:
		var active_slot = skillbar.get_node("GridContainer").get_child(active_slot_index)
		if active_slot and active_slot.associated_slot and active_slot.associated_slot.item:
			var item = active_slot.associated_slot.item
			var current_direction = determine_current_direction()
			if item.animations.has(current_direction):
				anim_player.play(item.animations[current_direction])
				
func _on_AnimationPlayer_animation_finished(anim_name: String) -> void:
	if anim_name == current_manual_animation:
		$woodAxe.set_monitoring(false)
		is_manual_animation_playing = false
		if last_vertical_direction == "down" and last_horizontal_direction != "":
			anim_player.play(last_horizontal_direction + "IdleAnim")
		elif last_vertical_direction == "up":
			anim_player.play("upIdleAnim")
		elif last_horizontal_direction !="":
			anim_player.play(last_horizontal_direction + "IdleAnim")
		
	if active_slot_index != -1:
		var active_slot = skillbar.get_node("GridContainer").get_child(active_slot_index)
		if active_slot and active_slot.associated_slot and active_slot.associated_slot.item:
				var item = active_slot.associated_slot.item
				if item.name == "shovel":
					replace_tile_with_sow()
					get_node("/root/World/Player/shovel")
				elif item.name == "woodAxe":
					$woodAxe.set_monitoring(false)
				item.is_being_used = false
				
	
func replace_tile_with_sow():
	var tile_coords = tile_map.local_to_map(highlightTileSprite.global_position)


	if tile_map.get_cell_source_id(0, tile_coords) == 1:
		tile_map.set_cell(0, tile_coords, 7, Vector2i(0, 0))

func water_tiles(num_tiles: int):
	print("Last Horizontal Direction: ", last_horizontal_direction)
	print("Last Vertical Direction: ", last_vertical_direction)
	
	
	var active_slot = skillbar.get_node("GridContainer").get_child(active_slot_index)
	if active_slot and active_slot.associated_slot and active_slot.associated_slot.item:
		var item = active_slot.associated_slot.item
		if item.name == "wateringCan":
			var tile_offset = Vector2i.ZERO
			if last_horizontal_direction == "left":
				tile_offset.x = -1
			if last_horizontal_direction == "right":
				tile_offset.x = 1
			if last_horizontal_direction == "up":
				tile_offset.x = -1
			if last_horizontal_direction == "down":
				tile_offset.x = 1
			if tile_offset.x != 0 and tile_offset.y != 0:
				tile_offset = Vector2i.ZERO
			var tile_size = Vector2(16, 16)
			var is_shift_pressed = Input.is_key_pressed(KEY_SHIFT)
			var tiles_to_process = 5 if is_shift_pressed else 1
			
			var tile_coords = tile_map.local_to_map(highlightTileSprite.global_position - tile_map.global_position)
			var tiles_to_water = []
			match last_movement_direction:
				"up":
					for i in range(4): 
						tiles_to_water.append(tile_coords + Vector2i(-1, -i - 1))
						tiles_to_water.append(tile_coords + Vector2i(0, -i - 1))
				"down":
					for i in range(4):
						tiles_to_water.append(tile_coords + Vector2i(-1, i + 1))
						tiles_to_water.append(tile_coords + Vector2i(0, i + 1))
				"left":
					for i in range(2):
						for j in range(5):
							tiles_to_water.append(tile_coords + Vector2i(-j - 1, -1 + i))
				"right":
					for i in range(2):
						for j in range(5):
							tiles_to_water.append(tile_coords + Vector2i(j + 1, -1 + i))
							
							
			print("Initial Tile Coordinates: ", tile_coords)
			print("Number of Tiles to Water: ", num_tiles)
			print("Tiles to Water: ", tiles_to_water)
			
			
			for i in range(min(num_tiles, len(tiles_to_water))):
				var tile_coord = tiles_to_water[i]
				if tile_map.get_cell_source_id(0, tile_coord) == SOW_TILE_ID:
				
					print("Watering Tile Number: ", i)
					print("Tile Coordinates: ", tile_coord)
					print("Tile Map Cell Source ID Before: ", tile_map.get_cell_source_id(0, tile_coord))
					tile_map.set_cell(0, tile_coord, WATERED_SOW_TILE_ID, Vector2i(0, 0))
					print("Tile Map Cell Source ID After: ", tile_map.get_cell_source_id(0, tile_coord))
		


func water_tile():
	if watering_can_water_level <= 0:
		print("Watering can is empty!")
		return
	
	var tile_coords = tile_map.local_to_map(highlightTileSprite.global_position - tile_map.global_position)
	if tile_map.get_cell_source_id(0, tile_coords) in [SOW_TILE_ID]:
		tile_map.set_cell(0, tile_coords, WATERED_SOW_TILE_ID, Vector2i(0, 0))
		watering_can_water_level -= 10
		var active_slot = skillbar.get_node("GridContainer").get_child(active_slot_index)
		if active_slot and active_slot.associated_slot and active_slot.associated_slot.item:
			var item = active_slot.associated_slot.item
			if item.name == "wateringCan":
				item.durability -= 10 

func position_highlight_tile():
	var tile_offset = Vector2i.ZERO
	
	
	if last_horizontal_direction == "left":
			tile_offset.x = -1
	elif last_horizontal_direction == "right":
			tile_offset.x = 1
	if last_vertical_direction == "up":
			tile_offset.y = -1
	elif last_vertical_direction == "down":
			tile_offset.y = 1
			
	
	if tile_offset.x != 0 and tile_offset.y != 0:
		tile_offset = Vector2i.ZERO
		
	var adjusted_offset = 32
	var adjusted_player_position = global_position + Vector2(tile_offset.x * adjusted_offset, tile_offset.y * adjusted_offset)
	var current_tile_coords = tile_map.local_to_map(global_position)
	var target_tile_coords = tile_map.local_to_map(adjusted_player_position)
	if current_tile_coords == target_tile_coords:
		target_tile_coords += tile_offset
	
	var tile_size = Vector2(16, 16)
	var tile_center_position = Vector2(target_tile_coords.x * tile_size.x + tile_size.x * 0.5, target_tile_coords.y * tile_size.y + tile_size.y * 0.5)
	var tile_source_id = tile_map.get_cell_source_id(0, target_tile_coords)
	if tile_source_id == GROUND_TILE_ID or tile_source_id == SOW_TILE_ID or tile_source_id == WATERED_SOW_TILE_ID:
		highlightTileSprite.global_position = tile_center_position + tile_map.global_position
	
	
	#print("Highlight Global Position: ", highlightTileSprite.global_position)
	#print("Player Global Position: ", global_position)
	#print("Current Tile Coords: ", current_tile_coords)
	#print("Target Tile Coords: ", target_tile_coords)

	#highlightTileSprite.global_position = global_position + tile_offset * highlightTileSprite.texture.get_size()




func _on_shovel_area_entered(area):
	pass #




func _on_wood_axe_area_entered(area: Area2D) -> void:
	print("woodAxe entered an area!")
	Global.axe_hit(area)
