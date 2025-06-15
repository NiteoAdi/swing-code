# swing-code
extends CharacterBody2D

const SPEED = 300.0
const JUMP_VELOCITY = -400.0
const MAX_ROPE_LENGTH = 1000.0

@onready var animation = $AnimatedSprite2D

var gravity = ProjectSettings.get_setting("physics/2d/default_gravity")
var hook_pos = Vector2()
var hooked = false
var current_rope_length = MAX_ROPE_LENGTH

func _physics_process(delta):
	# 🔹 Always check for hook first (even mid-air)
	hook()

	# 🔹 Apply gravity if not hooked
	if not is_on_floor() and not hooked:
		velocity.y += gravity * delta

	# 🔹 Handle jumping (only if on floor)
	if Input.is_action_just_pressed("ui_accept") and is_on_floor():
		velocity.y = JUMP_VELOCITY

	# 🔹 Horizontal movement
	var direction = Input.get_axis("ui_left", "ui_right")
	if direction != 0:
		velocity.x = direction * SPEED
		animation.flip_h = direction < 0
	else:
		velocity.x = move_toward(velocity.x, 0, SPEED)

	# 🔹 Animation handling
	if not is_on_floor():
		animation.play("jump")
	elif direction != 0:
		animation.play("run")
	else:
		animation.play("idle")

	# 🔹 Swinging logic overrides gravity
	if hooked:
		swing(delta)

	# 🔹 Movement (Godot 4 style)
	move_and_slide()

	# 🔹 Raycast optional visual/debug (future use)
	$Raycast.look_at(get_global_mouse_position())
	for raycast in $Raycast.get_children():
		if raycast.is_colliding():
			var _colliding_point = raycast.get_collision_point()
			# Optional: use _colliding_point

	queue_redraw()

func _draw():
	# 🔹 Draw rope line
	if hooked:
		draw_line(Vector2.ZERO, hook_pos - global_position, Color(0, 1, 0.3, 1), 3, true)

func hook():
	$Raycast.look_at(get_global_mouse_position())

	# 🔹 Try to hook when mouse is clicked
	if Input.is_action_just_pressed("left_click"):
		var result = get_hook_pos()
		if result != null:
			var dist = global_position.distance_to(result)
			if dist <= MAX_ROPE_LENGTH:
				hook_pos = result
				hooked = true
				current_rope_length = dist

	# 🔹 Release hook on button release
	if Input.is_action_just_released("left_click") and hooked:
		hooked = false

func get_hook_pos():
	# 🔹 Return first raycast that hits
	for raycast in $Raycast.get_children():
		if raycast.is_colliding():
			return raycast.get_collision_point()
	return null

func swing(delta):
	var radius = global_position - hook_pos
	var radius_len = radius.length()
	var vel_len = velocity.length()

	# 🔹 Avoid division issues
	if vel_len < 0.01 or radius_len < 10:
		return

	var dot = clamp(radius.dot(velocity) / (radius_len * vel_len), -1, 1)
	var angle = acos(dot)
	var radial_velocity = cos(angle) * vel_len

	# 🔹 Adjust velocity for realistic swing pull
	velocity += radius.normalized() * -radial_velocity

	# 🔹 Lock position within rope length
	if radius_len > current_rope_length:
		global_position = hook_pos + radius.normalized() * current_rope_length

	# 🔹 Apply hook pulling force
	velocity += (hook_pos - global_position).normalized() * 15000 * delta
