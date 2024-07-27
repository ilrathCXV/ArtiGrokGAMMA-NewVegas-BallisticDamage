===========================================================
== PLAYER-VS-ENEMY + ENEMY-VS-ENEMY* DAMAGE CALCULATIONS ==		* if you have the necessary MCM options enabled
===========================================================
This section will detail the logic that BALLISTIC/MELEE damage follows when (1) the player attacks a Stalker and (2) when a Stalker attacks another Stalker.
The system attempts to mimic the Fallout: New Vegas damage system, specifically pertaining to Damage Resistance (DR) and Damage Threshold (DT).

The damage calculations are split up into three parts: (1) Starter Damage, (2) Resist Damage, and (3) Final Base Damage. The variables will be pulled from how GAMMA/GBOOBS usually handles damage,
with some personal twists to better have it fit in GAMMA's vision of play.

//////////////////////////////
/////// Starter Damage ///////
//////////////////////////////
This will be the first calculation of damage that the rest of the formula will based on. This damage is essentially the raw damage of your damage source: your weapon.

This damage value is dervied from the following components:
	- Weapon Base Damage 		= wpn_hit_power
	- Weapon's Barrel Condition = barrel_condition_corrected
	- Bonus pre-modifiers such as:
		- Falloff modifier (will use GAMMA's or ArtiGrok/Momo's method depending on your MCM option)
		- Silencer boost (if the weapon is suppressed)
		
Here, I had to do a "necessary" evil: weapon damages in GAMMA for similar weapons in New Vegas are much higher. To try and balance things out
without making weapons turn into peashooters, I slightly nerfed the base damage of weapons on the fly.

Shotguns receive a 15% damage reduction, all others receive a 25% damage reduction. NPCs will always have a 25% reduction since they never use buckshot.

 	if is_actor then 
		if wpn_kind == "w_shotgun" then
			wpn_hit_power = wpn_hit_power * 0.85
		else
			wpn_hit_power = wpn_hit_power * 0.75
		end
	else
		wpn_hit_power = wpn_hit_power * 0.75
	end

Now, we plug in the needed values to get the Starter Damage:	
	
	starter_damage = wpn_hit_power * falloff_modifier * barrel_condition_corrected * silencer_boost
		
This final value will then be used for the next calculation step...

/////////////////////////////
/////// Resist Damage ///////
/////////////////////////////
This is the second calculation of damage where enemy DR, enemy DT, and ammo AP power comes into player.

	resist_damage = starter_damage

This damage value, which is initialized from the Starter Damage, takes these components into account:
	- DR (damage resistance) = custom_bone_ap_scale * Sin Res (multiplicative like New Vegas, capped at 85% DR/blocking 85% of damage)
	- DT (damage threshold) = custom_bone_armor (reduces damage by this amount after subtracted from k_ap)
	- AmmoDT = k_ap (ammo's AP power) & various adjusters from GAMMA (no change from normal GAMMA calcs) + any ammo effects modifying k_ap
	- AmmoDT_mult = [NOT USED] 0.80 (Grok: last multiplier is a new adjustment after the bone grouping update)
	- CritDMG = x2 if NPC is surrendering
	
This is where things stop being simple. The process to find the Resist Damage value is an involved process as we get closer to the end.

Firstly, we apply the "CritDMG"/Surrender Bonus that GAMMA gives. This is only applied if the enemy is in a surrendering state.
	
	resist_damage = resistance * 2.0
	
Secondly, we find the total multiplicative amount of DR the enemy has. This will always be capped at 85% (or a modifier of 15%).
For enemies, their DR rarely reaches that high, but it is necessary. DR can result in granting more damage if the victim has been given a "Sin Res"/faction resistance value higher than 1.0.

Here is a handy thing to remember in this case as these values aren't too clear cut to math gurus:

The higher the value, the less damage it blocks (anything above 1.0 will grant MORE damage to be caused).
The lower the value, the more damage it blocks.
Ammo with the Basher effect ore Headhunter effect tha land a headshot, they will skip this step.

	total_resist_modifier = custom_bone_ap_scale * sin_res
	total_resist_modifier = max(total_resist_modifier, 0.15) --> 0.15 = 85% DR | This ensures that if the total DR modifier goes below 0.15/the DR is above 85%, it gets capped out
	
	(isg_res is not included in here as that pertains to AP power, which is handled with DT calculations)
	
Next, we need to figure out the AP power of the ammo we fired. This is straight-forward as nothing about it really changed to how GAMMA finds the true AP power.
NOTE: GAMMA has ammo's k_ap values divided by 10 to reduce multi-hit situtations, so know that when we use k_ap here, it has already been multipled by 10 to get its true value back.

	k_ap = k_ap * falloff_modifier * isg_res * silencer_boost * ((is_actor and difficulty_multiplier[game_num]) or npc_difficulty_multiplier[game_num])
	
	(Difficulty calcuations are meant for Final Base Damage, but it would muddy further calculations)
	(Ammo special effects that modify k_ap have already been applied before any of the damage calcuations were done)
	
Once that has been solved for, we take the enemy's locational DT (custom_bone_armor) and your ammo's AP Power and compare the two to see how much DT will be left.
If there is still DT left over, that remaining amount will be subtracted from the damage amount, further reducing the damage amount.
An enemy's DT will NOT be reduced on non-penetrating hits., but it will be nullified on that hit location for 3 seconds if it is broken.
Determining if a limb's DT was broken/penetrated will be discussed in the next damage section.

	remaining_dmg_threshold = math.max(0, custom_bone_armor - k_ap)

Finally, we input all of the components and solve for the Resist Damage.
NOTE: Following New Vegas Logic, DR and DT together will NEVER cause Resist Damage to fall below 20% of the Starter Damage.

	resist_damage = math.max((starter_damage * 0.2), ((resist_damage * total_resist_modifier) - remaining_dmg_threshold))
	
/////////////////////////////////
/////// Final Base Damage ///////
/////////////////////////////////
These will be the final steps to determining your final base damage before feeding it into other things like Perk-Based Artefacts or other mods like RPG Redux that grant damage increases.

	final_base_damage = resist_damage

These will be the components that will make up the calculations for our final value:
	- LM (locational mulitplier) = (custom_bone_dmg_mult * 1.1) & sniper_bad_bone_shit_reduce (base GAMMA values)
	- DM (difficulty) =  difficulty_multiplier[game_num] ---> this will either do nothing or increase/reduce your damage based on your MCM value
	- Perks = PBA (hanlded outside of this function/mod), perk/RPG mods, etc.
	- AM (ammo multiplier) = k_hit & ammo_mult & hp_penality_modifier (1/hp_rounds) & all possible ammo effects (grav_mult, ambush_mult, chaos_mult, flinch_mult, etc.)
	
First, we check to see if the enemy is either suffering from the Armorbuster effect or being hit with an Acid round (Chaos Rounds can randomly trigger these as well).
We then use those values to further decreased the remaining DT the enemy has after the previous calculations.
	
	armorbuster_acid_effect_bonus = (armor_break_bonus * armor_break_mult) or 0
	new_remaining_threshold = remaining_dmg_threshold - armorbuster_acid_effect_bonus
	
We will then get to find any non-pen. penalties to rounds that suffer from them (i.e. most HP rounds). This is to discourage spamming incorrect ammo.
If no penalty exists OR the hit did penetrate the new remaining DT, this modifier will not affect damage.

		hp_penality_modifier = (1/hp_rounds) or 1

For the Locational Multiplier, it will be capped to 2.0 if the victim's DT was not destroyed/penetrated. This is mostly seen on headshots as it tends to be well above 2.0.

	total_locational_modifier = (custom_bone_dmg_mult * 1.1) * sniper_bad_bone_shit_reduce
	if new_remaining_threshold > 0 then
		total_locational_modifier = math.min(total_locational_modifier, 2.0)
	end

Currently, some weapons that weren't breaking DT/full pen., they would still do a good amount of damage to where you could spray an armored enemy with HP to win.
To counteract this, we take the remaining DT and apply it again, with the damage having a floor of 0.25 Damage (0.0025).

	if new_remaining_threshold > 0 then
		final_base_damage = math.max(0.0025, (final_base_damage - new_remaining_threshold))
	end

With the last needed variable that truly affects damage, we will plug what we can into the damage:

		total_ammo_mult = k_hit * ammo_mult * hp_penality_modifier * grav_mult * fragment_mult * ambush_mult * chaos_mult
		difficulty_modifier = ((is_actor and difficulty_multiplier[game_num]) or npc_difficulty_multiplier[game_num]) or 1
		
		final_base_damage = math.max(((0.01 * flinch_stun_chance)/pellet_count), final_base_damage * total_ammo_mult * total_locational_modifier * difficulty_modifier)
		
		(NOTE: Minimum damage will be set to (1 Damage * ammo's stun chance) divided by the pellet count. This will help prevent people from spamming buckshot to kill a target.)

Now, we will determine what to do with penetration hits.
If the new remaining DT is zero or less, the hit will count as penetrative, meaning the enemy's DT has been broken.
If an enemy had the DT bypassed and destroyed on the hit limb, there are a few things that happen with that:
	(1) The bonus from Armorbuster/Acid effects will be ADDED to your damage value. This is to give those effects more value than just post-DT reductions.
	
		final_base_damage = final_base_damage + armorbuster_acid_effect_bonus
	
	(2) That hit limb will have 0 DT for 5 seconds. After that, DT is back to its normal value.
		New Vegas had its broken DT last for 3 seconds, but, since we swap weapons mostly slower than in New Vegas, I extended it to 5 seconds.
		This means that you could swap to a weapon with good damage but bad AP power and do insane damage. This can give pistol and buckshot lovers more time in the limelight.
		If you break another limb, the timer will effectively be extended another 5 seconds.
		
		if new_remaining_threshold <= 0 then
		final_base_damage = final_base_damage + armorbuster_acid_effect_bonus
		-- ilrathCXV (07/26/24): If their bone's DT has been overcome, it will stay broken for 5 seconds
		custom_bone_value[custom_bone_id] = 0
		-- Ensures that the removal timer only occurs if active
		if not custom_bone_broken_dt[custom_bone_id] then
			custom_bone_broken_dt[custom_bone_id] = true
			CreateTimeEvent("reset_dmg_threshold"..custom_bone_id..math.random(1,100), npc:id(), 5.0, reset_dmg_threshold, custom_bone_id, custom_bone_armor)
		end
		
		[...]
		
	end

	(3) All hits to other limbs while the enemy is in a "broken DT" state will receive 3x more damage **IF** the Final Base Damage after the first batch of calculations for it
		are equal to or below 1 Damage (0.01). If the damage is still far below 1 Damage, it will be brought up to that.
		
		if new_remaining_threshold <= 0 then
		
			[...]
		
			-- Ensures that if enemy had another DT broken, it doesn't immediately take it away when the first DT break is up
			npc_broken_dt[npc:id()] = (npc_broken_dt[npc:id()] and npc_broken_dt[npc:id()] + 1) or 1
			CreateTimeEvent("reduce_npc_broken"..npc:id()..math.random(1,100), npc:id(), 5.0, reduce_npc_broken, npc:id())
		end
		
		
		if npc_broken_dt[npc:id()] then
			if npc_broken_dt[npc:id()] > 0 then
				printf('Enemy DT is broken! Checking if low DMG needs to be increased...')
				if final_base_damage <= 0.01 then
					final_base_damage = math.max(0.01, final_base_damage * broken_dt_modifier)
				end
			end
		end
		
With all of that, you are have your new Final Base Damage, which will then be assigned to the original hit (shit) and sent off to other functions and mods like PBA.

	shit.power = final_base_damage
	
This is the conclusion of the New Vegas "PvS" and "SvS" documentation.