# Assasin

# Залить

    git add .
    git commit -m "Тут комментарий"
    git push

# Слить

    git pull


-- Player logic

-- these are the tweaks for the mechanics, feel free to change them for a different feeling
-- max speed right/left
local MAX_GROUND_SPEED = 250
local MAX_AIR_SPEED = 200
-- max fall speed
local MAX_FALL_SPEED = 500

-- gravity pulling the player down in pixel units
local GRAVITY = -500
-- take-off speed when jumping in pixel units
local JUMP_TAKEOFF_SPEED = 350
local DOUBLEJUMP_TAKEOFF_SPEED = 200
-- push-back when shooting
local RECOIL = 500

-- pre-hashing ids improves performance
local CONTACT_POINT_RESPONSE = hash("contact_point_response")
local GROUND = hash("ground")
local RESPAWMN = hash("respawn")
local ENEMY = hash("enemy")

local LEFT = hash("Left")
local RIGHT = hash("Right")
local JUMP = hash("Jump")
--local ATTACK = 

local ANIM_WALK = hash("Run")
local ANIM_IDLE = hash("Idle")
local ANIM_JUMP = hash("Jump")
local ANIM_FALL = hash("Idle")
local ANIM_ATTACK = hash("Attack")

local SPRITE = "#Player"
local isAttacking=false

function init(self)
	-- this lets us handle input in this script
	msg.post(".", "acquire_input_focus")

	-- activate camera attached to the player collection
	-- this will send camera updates to the render script
	-- msg.post("camera", "acquire_camera_focus")

	-- spawn position
	self.spawn_position = go.get_position()
	-- player velocity
	self.velocity = vmath.vector3(0, 0, 0)
	-- which direction the player is facing
	self.direction = 1
	-- support variable to keep track of collisions and separation
	self.correction = vmath.vector3()
	-- if the player stands on ground or not
	self.ground_contact = true
	-- also track state of last frame
	-- (to detect when landing or taking off)
	self.previous_ground_contact = true
	-- the currently playing animation
	self.anim = nil
end

function endofanimation(self)
	if self.anim ~= anim then
		isAttacking=false
	end
end

local function play_animation(self, anim)
	-- only play animations which are not already playing
	if self.anim ~= anim then
		-- tell the sprite to play the animation
		sprite.play_flipbook("#Player", anim, endofanimation)
		-- remember which animation is playing
		self.anim = anim
	end
end

local function squish()
	go.animate("#Player", "scale.x", go.PLAYBACK_ONCE_PINGPONG, 0.8, go.EASING_INOUTQUAD, 0.6)
	go.animate("#Player", "scale.y", go.PLAYBACK_ONCE_PINGPONG, 1.2, go.EASING_INOUTQUAD, 0.6)
end

local function update_animations(self)
	-- make sure the player character faces the right way
	sprite.set_hflip(SPRITE, self.direction == -1)

	-- make sure the right animation is playing
	if self.ground_contact then
		if self.velocity.x == 0 then
			play_animation(self, ANIM_IDLE)
		else
			play_animation(self, ANIM_WALK)
		end
	else
		if self.velocity.y > 0 then
			play_animation(self, ANIM_JUMP)
		else
			play_animation(self, ANIM_FALL)
		end
	end
end

-- clamp a number between a min and max value
local function clamp(v, min, max)
	if v < min then return min
	elseif v > max then return max
	else return v end
end

-- apply an opposing force to decrease a velocity
local function decelerate(v, f, dt)
	local opposing = math.abs(v * f)
	if v > 0 then
		return math.floor(math.max(0, v - opposing * dt))
	elseif v < 0 then
		return math.ceil(math.min(0, v + opposing * dt))
	else
		return 0
	end
end

function update(self, dt)
	-- apply gravity
	self.velocity.y = self.velocity.y + GRAVITY * dt
	self.velocity.y = clamp(self.velocity.y, -MAX_FALL_SPEED, MAX_FALL_SPEED)

	-- apply ground or air friction
	if self.ground_contact then
		self.velocity.x = decelerate(self.velocity.x, 20, dt)
		self.velocity.x = clamp(self.velocity.x, -MAX_GROUND_SPEED, MAX_GROUND_SPEED)
	else
		self.velocity.x = decelerate(self.velocity.x, 1, dt)
		self.velocity.x = clamp(self.velocity.x, -MAX_AIR_SPEED, MAX_AIR_SPEED)
	end

	-- move player
	local pos = go.get_position()
	pos = pos + self.velocity * dt
	go.set_position(pos)

	-- update animations based on state (ground, air, move and idle)
	update_animations(self)

	-- reset volatile state
	self.previous_ground_contact = self.ground_contact
	self.correction = vmath.vector3()
	self.ground_contact = false
	self.wall_contact = false
end

local NORMAL_THRESHOLD = 0.7

-- https://defold.com/manuals/physics/#resolving-kinematic-collisions
local function handle_obstacle_contact(self, normal, distance)
	-- don't care about anything but normals beyond the threshold
	-- we do this to eliminate false-positives such as ceiling hits when
	-- jumping next to a wall while moving into the wall
	if normal.y < NORMAL_THRESHOLD and normal.y > -NORMAL_THRESHOLD then
		normal.y = 0
	end
	if normal.x < NORMAL_THRESHOLD and normal.x > -NORMAL_THRESHOLD then
		normal.x = 0
	end
	-- update distance in case the normals have changed
	distance = distance * vmath.length(normal)

	if distance > 0 then
		-- First, project the accumulated correction onto
		-- the penetration vector
		local proj = vmath.project(self.correction, normal * distance)
		if proj < 1 then
			-- Only care for projections that does not overshoot.
			local comp = (distance - distance * proj) * normal
			-- Apply compensation
			go.set_position(go.get_position() + comp)
			-- Accumulate correction done
			self.correction = self.correction + comp
		end
	end

	-- collided with a wall
	-- stop horizontal movement
	if math.abs(normal.x) > NORMAL_THRESHOLD then
		self.wall_contact = true
		self.velocity.x = 0
	end
	-- collided with the ground
	-- stop vertical movement
	if normal.y > NORMAL_THRESHOLD then
		if not self.previous_ground_contact then
			-- add some particles 
			-- particlefx.play("#dust")
			-- reset any "squish" that may have been applied
			go.set("Player", "scale", 1)
			self.double_jump = false
		end
		self.ground_contact = true
		self.velocity.y = 0
	end
	-- collided with the ceiling
	-- stop vertical movement
	if normal.y < -NORMAL_THRESHOLD then
		self.velocity.y = 0
	end
end

function on_message(self, message_id, message, sender)
	-- check if we received a contact point message
	if message_id == CONTACT_POINT_RESPONSE then
		-- check that the object is something we consider an obstacle
		if message.group == GROUND then
			handle_obstacle_contact(self, message.normal, message.distance)
		elseif message.group == RESPAWMN or message.group == ENEMY then
			go.set_position(self.spawn_position)
		end
	end
end

local function jump(self)
	-- only allow jump from ground
	-- (extend this with a counter to do things like double-jumps)
	if self.ground_contact then
		-- set take-off speed
		self.velocity.y = JUMP_TAKEOFF_SPEED
		-- play animation
		play_animation(self, ANIM_JUMP)
		self.ground_contact = false
		-- compress and stretch player for visual "juice"
		squish()
		-- allow double jump if still moving up
	elseif not self.double_jump and self.velocity.y > 0 then
		self.velocity.y = self.velocity.y + DOUBLEJUMP_TAKEOFF_SPEED
		self.double_jump = true
	end
	-- add some particles 
	-- particlefx.play("#Jump")
end

local function abort_jump(self)
	-- cut the jump short if we are still going up
	if self.velocity.y > 0 then
		-- scale down the upwards speed
		self.velocity.y = self.velocity.y * 0.5
	end
end

local function walk(self, direction)
	if direction ~= 0 then
		self.direction = direction
	end
	if self.ground_contact then
		self.velocity.x = MAX_GROUND_SPEED * direction
	else
		-- move slower in the air
		self.velocity.x = MAX_AIR_SPEED * direction
	end
end

local function fire_knife(self)
	self.velocity.x = self.velocity.x + RECOIL * -self.direction

	local pos = go.get_position()
	-- offset the bullet so that it is fired "outside" of the player sprite
	pos.x = pos.x + 8 * self.direction
	-- local id = factory.create("#bulletfactory", pos)

	-- flip the bullet sprite
	sprite.set_hflip(msg.url(nil, id, "sprite"), self.direction == -1)

	-- move the bullet 300 pixels and then delete it
	local distance = 150
	local to = pos.x + distance * self.direction
	local duration = distance / 250
	go.animate(id, "position.x", go.PLAYBACK_ONCE_FORWARD, to, go.EASING_LINEAR, duration, 0, function()
		go.delete(id)
	end)
end

local function anim_attack(self)

	play_mya(self, ANIM_ATTACK)
end

function on_input(self, action_id, action)
	if action_id == LEFT then
		walk(self, -action.value)
	elseif action_id == RIGHT then
		walk(self, action.value)
	elseif action_id == JUMP then
		if action.pressed then
			jump(self)
		elseif action.released then
			abort_jump(self)
		end
	elseif action_id == ANIM_ATTACK then
		self.isAttacking=true
	end
end



1111111111111111111111111111

local MAX_GROUND_SPEED = 250
local MAX_AIR_SPEED = 200
local MAX_FALL_SPEED = 500
local GRAVITY = -500
local JUMP_TAKEOFF_SPEED = 350
local DOUBLEJUMP_TAKEOFF_SPEED = 200
local RECOIL = 500

local CONTACT_POINT_RESPONSE = hash("contact_point_response")
local GROUND = hash("ground")
local RESPAWMN = hash("respawn")
local ENEMY = hash("enemy")
local LEFT = hash("Left")
local RIGHT = hash("Right")
local JUMP = hash("Jump")

local ANIM_WALK = hash("Run")
local ANIM_IDLE = hash("Idle")
local ANIM_JUMP = hash("Jump")
local ANIM_FALL = hash("Idle")
local ANIM_ATTACK = hash("Attack")

local IDLE = hash("STATE_IDLE")
local WALK = hash("STATE_WALK")
local JUMP = hash("STATE_JUMP")
local ATTACK = hash("STATE_ATTACK")


local isAttacking = false

local state, previous_state

function init(self)
	msg.post(".", "acquire_input_focus")

	self.spawn_position = go.get_position()
	self.velocity = vmath.vector3(0, 0, 0)
	self.direction = 1
	self.correction = vmath.vector3()
	self.ground_contact = true
	self.previous_ground_contact = true
	self.anim = nil
	self.state = IDLE
	self.previous_state = nil
end

function endofanimation(self)
	if self.anim ~= anim then
		isAttacking=false
	end
end

local function play_animation(self, anim)
	if self.anim ~= anim then
		sprite.play_flipbook("#Player", anim, endofanimation)
		self.anim = anim
	end
end

local function squish()
	go.animate("#Player", "scale.x", go.PLAYBACK_ONCE_PINGPONG, 0.8, go.EASING_INOUTQUAD, 0.6)
	go.animate("#Player", "scale.y", go.PLAYBACK_ONCE_PINGPONG, 1.2, go.EASING_INOUTQUAD, 0.6)
end

local function update_animations(self)
	sprite.set_hflip(SPRITE, self.direction == -1)

	if self.ground_contact then
		if self.velocity.x == 0 then
			play_animation(self, ANIM_IDLE)
		else
			play_animation(self, ANIM_WALK)
		end
	else
		if self.velocity.y > 0 then
			play_animation(self, ANIM_JUMP)
		else
			play_animation(self, ANIM_FALL)
		end
	end
end

function ChangeState(self, state)
	if (state == IDLE) then
		if self.move_input ~= 0 then
			self.previous_state = state 
			self.state = WALK
		end
	elseif state == WALK then
		if self.move_input == 0 then
			self.previous_state = state 
			self.state = IDLE
		end
	end
end

function ExecuteState(self, state)
	if state == WALK then
		label.set_text("Debug#Text", "WALK")
		update_animation(self)
	elseif state == IDLE then
		label.set_text("Debug#Text", "IDLE")
		play_animation(self, anim_idle)
	end
end

local function clamp(v, min, max)
	if v < min then return min
	elseif v > max then return max
	else return v end
end

local function decelerate(v, f, dt)
	local opposing = math.abs(v * f)
	if v > 0 then
		return math.floor(math.max(0, v - opposing * dt))
	elseif v < 0 then
		return math.ceil(math.min(0, v + opposing * dt))
	else
		return 0
	end
end

function update(self, dt)
	self.velocity.y = self.velocity.y + GRAVITY * dt
	self.velocity.y = clamp(self.velocity.y, -MAX_FALL_SPEED, MAX_FALL_SPEED)

	if self.ground_contact then
		self.velocity.x = decelerate(self.velocity.x, 20, dt)
		self.velocity.x = clamp(self.velocity.x, -MAX_GROUND_SPEED, MAX_GROUND_SPEED)
	else
		self.velocity.x = decelerate(self.velocity.x, 1, dt)
		self.velocity.x = clamp(self.velocity.x, -MAX_AIR_SPEED, MAX_AIR_SPEED)
	end

	local pos = go.get_position()
	pos = pos + self.velocity * dt

	ExecuteState(self, self.state)
	
	go.set_position(pos)

	update_animations(self)

	self.previous_ground_contact = self.ground_contact
	self.correction = vmath.vector3()
	self.ground_contact = false
	self.wall_contact = false
end

local NORMAL_THRESHOLD = 0.7

local function handle_obstacle_contact(self, normal, distance)
	if normal.y < NORMAL_THRESHOLD and normal.y > -NORMAL_THRESHOLD then
		normal.y = 0
	end
	if normal.x < NORMAL_THRESHOLD and normal.x > -NORMAL_THRESHOLD then
		normal.x = 0
	end
	distance = distance * vmath.length(normal)

	if distance > 0 then
		local proj = vmath.project(self.correction, normal * distance)
		if proj < 1 then
			local comp = (distance - distance * proj) * normal
			go.set_position(go.get_position() + comp)
			self.correction = self.correction + comp
		end
	end

	if math.abs(normal.x) > NORMAL_THRESHOLD then
		self.wall_contact = true
		self.velocity.x = 0
	end
	if normal.y > NORMAL_THRESHOLD then
		if not self.previous_ground_contact then
			go.set("Player", "scale", 1)
			self.double_jump = false
		end
		self.ground_contact = true
		self.velocity.y = 0
	end
	if normal.y < -NORMAL_THRESHOLD then
		self.velocity.y = 0
	end
end

function on_message(self, message_id, message, sender)
	if message_id == CONTACT_POINT_RESPONSE then
		if message.group == GROUND then
			handle_obstacle_contact(self, message.normal, message.distance)
		elseif message.group == RESPAWMN or message.group == ENEMY then
			go.set_position(self.spawn_position)
		end
	end
end

local function jump(self)
	if self.ground_contact then
		self.velocity.y = JUMP_TAKEOFF_SPEED
		play_animation(self, ANIM_JUMP)
		self.ground_contact = false
		squish()
	elseif not self.double_jump and self.velocity.y > 0 then
		self.velocity.y = self.velocity.y + DOUBLEJUMP_TAKEOFF_SPEED
		self.double_jump = true
	end
end

local function abort_jump(self)
	if self.velocity.y > 0 then
		self.velocity.y = self.velocity.y * 0.5
	end
end

local function walk(self, direction)
	if direction ~= 0 then
		self.direction = direction
	end
	if self.ground_contact then
		self.velocity.x = MAX_GROUND_SPEED * direction
	else
		-- move slower in the air
		self.velocity.x = MAX_AIR_SPEED * direction
	end
end

local function fire_knife(self)
	self.velocity.x = self.velocity.x + RECOIL * -self.direction

	local pos = go.get_position()
	pos.x = pos.x + 8 * self.direction

	sprite.set_hflip(msg.url(nil, id, "sprite"), self.direction == -1)

	local distance = 150
	local to = pos.x + distance * self.direction
	local duration = distance / 250
	go.animate(id, "position.x", go.PLAYBACK_ONCE_FORWARD, to, go.EASING_LINEAR, duration, 0, function()
		go.delete(id)
	end)
end

local function anim_attack(self)

	play_mya(self, ANIM_ATTACK)
end

function on_input(self, action_id, action)
	if action_id == LEFT then
		walk(self, -action.value)
	elseif action_id == RIGHT then
		walk(self, action.value)
	elseif action_id == JUMP then
		if action.pressed then
			jump(self)
		elseif action.released then
			abort_jump(self)
		end
	elseif action_id == ANIM_ATTACK then
		self.isAttacking=true
	end
end