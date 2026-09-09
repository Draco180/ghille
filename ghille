--[[
================================================================================
  GHOST CHECK - read the ghillie crates without going there
================================================================================
  Your idea, and it gets around everything that blocked us:

    weld to a vehicle's driver seat (above it, not in it)
      -> pivot the VEHICLE to crash site 1; you are welded, so you go with it
      -> the crate streams in and we read whether it is empty
      -> pivot to crash site 2, same
      -> unweld: you snap back home, which is fine - you never wanted to be
         there unless there was loot

  WHY THIS WORKS WHERE EVERYTHING ELSE FAILED
    We stopped needing the server to agree with us. Chunk streaming is done by
    YOUR client, off YOUR client's position, so if your client thinks it is at
    the crash site then the loot models and their state tables get built - and
    that is all the check ever read. The server can carry on believing you are
    at home.

    The weld is what makes it hold: welded to an anchored part you stay put,
    where a bare character teleport gets corrected within about a second - far
    too fast for anything to stream.

    The vehicle does not need to be occupied, does not need to survive, and
    cannot explode, because none of this is real from the server's point of
    view.

  VERIFIED WORKING (2026-09-05)
    The open question was whether the state was real or a default for crates
    the server never told us about. Reading EVERY crate at a site settled it:

        SeahawkCrashsite01 - 10 crates, 10 read: 5 empty, 5 with loot
        SeahawkCrashsite03 - 11 crates, 11 read: 7 empty, 4 with loot

    21 of 21 read, different crates full at each site. A default would have
    come back uniform. The read is genuine.

  RUNNING IT HANDS-OFF
    With AUTO_HOP on it hops by itself when both ghillie crates are empty. To
    make it survive the hop, save this file into your executor's autoexec
    folder - it checks the PlaceId and does nothing in other games. Otherwise
    just re-execute it after each hop.

  BUTTONS: CHECK runs a pass. HOP changes server. CLEAR unwelds you if stuck.
================================================================================
]]

local CONFIG = {
    SEAT_LIFT    = 2,      -- studs above the seat, so you are not inside it
    LIFT         = 2,      -- clearance above the site's ground. keep both small:
                           -- whatever height you are at when the weld comes off
                           -- is the height you fall from.
    LAND_SOFT    = 2.5,    -- seconds of zeroed velocity after unwelding, so the
                           -- warp home cannot land you at speed
    APPROACH     = 30,     -- stop this far short of the site

    -- waiting. the crate existing is NOT the same as the crate being loaded.
    MIN_DWELL    = 4,      -- always sit here this long, however fast it looks
    HOLD_SECS    = 25,     -- give up on a site after this
    STABLE_SECS  = 2,      -- site must stop gaining LootGroups for this long
    POLL         = 0.5,    -- seconds between state reads (getgc is expensive)
    READ_TRIES   = 5,      -- re-read this many times before calling it unknown
    AUTO_HOP     = true,   -- verified working, so hop by itself when empty
    EXPECTED_SITES = 2,    -- refuse to hop unless this many sites were checked,
                           -- so a missing site can never be silently skipped
    QUEUE_ON_HOP = true,   -- resume after a hop without re-pasting the script.
                           -- Set to FALSE if you ever put this in autoexec: the
                           -- autoexec copy already runs on join, and queueing
                           -- as well starts two copies that fight each other.
    AUTO_START   = true,   -- run a check on join (for autoexec use). Turn this
                           -- OFF if you ever die and want to recover a corpse -
                           -- a check ends in a hop, and the corpse only exists
                           -- in the server you died in.
    START_DELAY  = 6,      -- minimum wait before the first check
    HOP_PAGES    = 8,      -- pages of 100 servers to pull, per sort order.
    KEEP_RECENT  = 60,     -- when the list runs dry, forget all but this many
                           -- recent servers. Roblox recycles JobIds, so an old
                           -- one is usually a different server by the time we
                           -- come back to it.
    VISITED_FILE = "ar2_visited_servers.txt",
}

local GHILLIE_SITES = {
    SeahawkCrashsite01 = true, SeahawkCrashsite02 = true,
    SeahawkCrashsite03 = true, SeahawkCrashsite04 = true,
}
local GHILLIE_CRATE = "CrateShippingWoodMilitary02"
local GHILLIE_TABLE = "Clothing Unique GhillieGreen"
local WELD_NAME     = "AR2GhostWeld"

-- safe to sit in an autoexec folder: it does nothing in any other game
if game.PlaceId ~= 863266079 then return end

-- Only ONE copy of this may run at a time. The autoexec/queue_on_teleport copy
-- and a manual execution can both start, and then two passes fight over the
-- same vehicle - pivoting it to different sites, so nothing streams and a site
-- reads 0 crates. Each load takes the next generation number; older loops see
-- they are stale and stop.
local ENV = (getgenv and getgenv()) or _G
ENV.AR2_GHOST_GEN = (ENV.AR2_GHOST_GEN or 0) + 1
local MY_GEN = ENV.AR2_GHOST_GEN
local function stale() return ENV.AR2_GHOST_GEN ~= MY_GEN end

local Players = game:GetService("Players")
local RS      = game:GetService("ReplicatedStorage")
local RunSvc  = game:GetService("RunService")
local TS      = game:GetService("TeleportService")
local HttpSvc = game:GetService("HttpService")
local LP      = Players.LocalPlayer

--==============================================================================
-- basics
--==============================================================================
local function hrp()
    local ch = LP.Character
    return ch and (ch:FindFirstChild("HumanoidRootPart") or ch.PrimaryPart)
end
local function hum()
    local ch = LP.Character
    return ch and ch:FindFirstChildWhichIsA("Humanoid")
end
local function myPos()
    local r = hrp()
    if r then local ok, p = pcall(function() return r.Position end) if ok then return p end end
end

local function clearWeld(quiet)
    local n, ch = 0, LP.Character
    if ch then
        pcall(function()
            for _, d in ipairs(ch:GetDescendants()) do
                if d.Name == WELD_NAME then d:Destroy() n = n + 1 end
            end
        end)
    end
    local h = hum()
    if h then
        pcall(function()
            h.PlatformStand = false
            h:ChangeState(Enum.HumanoidStateType.GettingUp)
        end)
    end

    -- FALL DAMAGE: unwelding drops you from the hold height, so you start
    -- falling, and the anti-cheat then warps you home WITH that downward
    -- velocity still on you - you land at speed and take the damage. Pin the
    -- velocity to zero for a moment so there is no fall to be hurt by.
    if n > 0 then
        task.spawn(function()
            local t0 = os.clock()
            while os.clock() - t0 < CONFIG.LAND_SOFT do
                local r = hrp()
                if not r then break end
                pcall(function()
                    r.AssemblyLinearVelocity = Vector3.new()
                    r.AssemblyAngularVelocity = Vector3.new()
                end)
                local hh = hum()
                if hh then
                    pcall(function() hh:ChangeState(Enum.HumanoidStateType.Landed) end)
                end
                RunSvc.Heartbeat:Wait()
            end
        end)
    end

    if not quiet then print(("[ghost] cleared %d weld(s)"):format(n)) end
    return n
end

--==============================================================================
-- the ride: any vehicle will do, occupied or not, it is never really moved
--==============================================================================
-- Prefer an ANCHORED vehicle. Unoccupied vehicles are anchored, so pivoting
-- one is a local ghost and costs nothing. A vehicle you are sitting in - or
-- just climbed out of - is unanchored and owned by you, so pivoting THAT is a
-- real move and blows it up. That is how the ATV died.
local function isAnchoredVehicle(v)
    local anchored, total = 0, 0
    pcall(function()
        for _, d in ipairs(v:GetDescendants()) do
            if d:IsA("BasePart") then
                total = total + 1
                if d.Anchored then anchored = anchored + 1 end
                if total > 25 then return end   -- a sample is enough
            end
        end
    end)
    return total > 0 and anchored == total
end

local function nearestVehicle()
    local me = myPos()
    if not me then return nil, "no character" end
    local folder = workspace:FindFirstChild("Vehicles")
    if not folder then return nil, "no workspace.Vehicles" end

    local bestAnchored, bdA, bestAny, bdB
    for _, v in ipairs(folder:GetChildren()) do
        local ok, cf = pcall(function() return v:GetPivot() end)
        if ok and cf then
            local d = (cf.Position - me).Magnitude
            if not bdB or d < bdB then bestAny, bdB = v, d end
            if isAnchoredVehicle(v) and (not bdA or d < bdA) then
                bestAnchored, bdA = v, d
            end
        end
    end

    if bestAnchored then return bestAnchored, bdA end
    if bestAny then
        warn("[ghost] no anchored vehicle nearby - using " .. bestAny.Name ..
             ", which may be destroyed by the move")
        return bestAny, bdB
    end
    return nil, "no vehicles"
end

local function seatOf(veh)
    local driver, any
    pcall(function()
        for _, d in ipairs(veh:GetDescendants()) do
            if d:IsA("VehicleSeat") or d:IsA("Seat") then
                any = any or d
                if d.Name == "Driver" then driver = d return end
            end
        end
    end)
    return driver or any
end

local function attach(veh)
    local seat = seatOf(veh)
    local part = seat
    if not part then
        pcall(function()
            for _, d in ipairs(veh:GetDescendants()) do
                if d:IsA("BasePart") then part = d return end
            end
        end)
    end
    local r = hrp()
    if not r or not part then return nil, "nothing to weld to" end

    local h = hum()
    -- PlatformStand ON here: we are not trying to sit, just to be carried
    if h then pcall(function() h.PlatformStand = true end) end

    local weld = Instance.new("Weld")
    weld.Name   = WELD_NAME
    weld.Part0  = r
    weld.Part1  = part
    weld.C0     = CFrame.new()
    weld.C1     = CFrame.new(0, CONFIG.SEAT_LIFT, 0)
    weld.Parent = r
    return weld, part.Name
end

local function moveVehicle(veh, pos)
    local cf = CFrame.new(pos)
    return (pcall(function()
        if typeof(veh.PivotTo) == "function" then veh:PivotTo(cf)
        elseif veh.PrimaryPart then veh:SetPrimaryPartCFrame(cf) end
    end))
end

--==============================================================================
-- sites and crate state (unchanged from the working checker)
--==============================================================================
local function ghillieSites()
    local sites = {}
    local ck = RS:FindFirstChild("Chunking")
    local cd = ck and ck:FindFirstChild("Chunk Data")
    if not cd then return sites end
    for _, chunk in ipairs(cd:GetChildren()) do
        for _, v in ipairs(chunk:GetChildren()) do
            if GHILLIE_SITES[v.Name] then
                local p
                pcall(function()
                    local val = v.Value
                    if typeof(val) == "CFrame" then p = val.Position end
                end)
                if p then
                    -- the same site can be listed under more than one chunk,
                    -- so drop duplicates by position or we check one twice and
                    -- never look at the other one
                    local dupe = false
                    for _, s in ipairs(sites) do
                        -- 40, not 100: two genuinely separate crash sites can
                        -- sit fairly close, and merging them means only one is
                        -- ever checked. Only fold together entries that are
                        -- plainly the same site listed under two chunks.
                        if (s.pos - p).Magnitude < 40 then dupe = true break end
                    end
                    if not dupe then sites[#sites + 1] = { name = v.Name, pos = p } end
                end
            end
        end
    end
    local me = myPos()
    if me then
        table.sort(sites, function(a, b)
            return (a.pos - me).Magnitude < (b.pos - me).Magnitude
        end)
    end
    return sites
end

-- returns the ghillie LootGroup, plus how many LootGroups the site has loaded
-- so far. the count is what tells us whether the site is still streaming in.
-- Find loot groups by POSITION, not by name.
--
-- Two distinct crash sites can share a name (seen in the logs: two different
-- SeahawkCrashsite04 holders with different contents). FindFirstChild returns
-- only the first, so a name lookup silently read one site twice and never
-- looked at the other - which would make a live crate invisible.
local SITE_RADIUS = 260   -- studs around the site centre to count as "here"

local function groupsNear(pos)
    local map = workspace:FindFirstChild("Map")
    local el  = map and map:FindFirstChild("Elements")
    local list, ghillie = {}, nil
    if not el then return list, nil end

    local function walk(inst, d)
        if d > 12 then return end
        for _, c in ipairs(inst:GetChildren()) do
            if c.Name == "LootGroup" then
                local part = c:FindFirstChild("BasePart") or c:FindFirstChildWhichIsA("BasePart")
                local okP, p = pcall(function() return part and part.Position end)
                if okP and p and (p - pos).Magnitude <= SITE_RADIUS then
                    -- Identify the crate two ways, and accept either.
                    --   strict: right model name AND the ghillie loot table
                    --   loose:  right model name alone
                    -- The loot table child is part of the map definition, so it
                    -- is there whether the crate is full or empty - but if the
                    -- live clone ever loses it we would stop recognising the
                    -- crate at all. At a crash site the only
                    -- CrateShippingWoodMilitary02 present IS the ghillie one,
                    -- so the loose match costs nothing and removes that risk.
                    local label, isGhillie, strict = "?", false, false
                    for _, k in ipairs(c:GetChildren()) do
                        if k:IsA("Model") then
                            label = k.Name
                            if k.Name == GHILLIE_CRATE then
                                isGhillie = true
                                if k:FindFirstChild(GHILLIE_TABLE) then strict = true end
                            end
                        end
                    end
                    list[#list + 1] = {
                        group = c, label = label, ghillie = isGhillie,
                        strict = strict, dist = (p - pos).Magnitude,
                    }
                    if isGhillie and not ghillie then ghillie = c end
                end
            end
            walk(c, d + 1)
        end
    end
    walk(el, 0)
    return list, ghillie
end

local function ghillieGroup(pos)
    local list, gh = groupsNear(pos)
    return gh, #list
end

-- every LootGroup at a site, found by position for the same reason as above
local function siteGroups(pos)
    return (groupsNear(pos))
end

-- ONE getgc pass reads every crate at the site. Reading them all is the test:
-- genuine data looks like a mix, a default looks like every crate agreeing.
local function readAll(list)
    if type(getgc) ~= "function" then return 0, "no getgc" end
    local ok, objs = pcall(getgc, true)
    if not ok or type(objs) ~= "table" then return 0, "getgc failed" end

    local want = {}
    for _, e in ipairs(list) do want[e.group] = e end

    local got = 0
    for _, o in ipairs(objs) do
        if type(o) == "table" then
            pcall(function()
                local e = want[rawget(o, "Instance")]
                if not e or e.empty ~= nil then return end
                if rawget(o, "IsEmpty") ~= nil then
                    e.empty  = rawget(o, "IsEmpty")
                    e.prompt = rawget(o, "Prompt")
                    got = got + 1
                    return
                end
                for _, v in pairs(o) do
                    if type(v) == "table" and rawget(v, "IsEmpty") ~= nil then
                        e.empty  = rawget(v, "IsEmpty")
                        e.prompt = rawget(v, "Prompt")
                        got = got + 1
                        return
                    end
                end
            end)
        end
    end
    return got
end


--==============================================================================
-- server hop
--==============================================================================
local function httpGet(url)
    local req = (syn and syn.request) or (http and http.request) or http_request or request
    if req then
        local ok, res = pcall(req, { Url = url, Method = "GET" })
        if ok and res and res.Body then return res.Body end
    end
    local ok, body = pcall(function() return game:HttpGet(url, true) end)
    if ok then return body end
end

-- Server memory. This is the difference between hopping through new servers and
-- bouncing between the same three forever, so it reports rather than assuming:
-- appendfile is missing in some executors and fails SILENTLY, which is exactly
-- how a "why do I keep landing in the same lobby" bug hides.
local function loadVisited()
    local visited, n = {}, 0
    pcall(function()
        if isfile and readfile and isfile(CONFIG.VISITED_FILE) then
            for id in tostring(readfile(CONFIG.VISITED_FILE)):gmatch("[^\r\n]+") do
                if id ~= "" and not visited[id] then
                    visited[id] = true
                    n = n + 1
                end
            end
        end
    end)
    return visited, n
end

local function saveVisited(id)
    -- try append, then fall back to read-modify-write, then report failure
    local ok = pcall(function()
        if appendfile then appendfile(CONFIG.VISITED_FILE, id .. "\n") end
    end)
    if ok and appendfile then return true end
    local ok2 = pcall(function()
        if not writefile then error("no writefile") end
        local prev = ""
        if isfile and readfile and isfile(CONFIG.VISITED_FILE) then
            prev = tostring(readfile(CONFIG.VISITED_FILE))
        end
        writefile(CONFIG.VISITED_FILE, prev .. id .. "\n")
    end)
    return ok2
end

-- prove the memory actually persists, rather than hoping it does
local function checkMemory()
    local _, before = loadVisited()
    local probe = "selftest_" .. tostring(math.random(1e6, 9e6))
    local wrote = saveVisited(probe)
    local after2, after = loadVisited()
    local works = wrote and after2[probe] == true
    if works then
        -- tidy the probe back out
        pcall(function()
            if writefile and readfile and isfile and isfile(CONFIG.VISITED_FILE) then
                local body = tostring(readfile(CONFIG.VISITED_FILE))
                writefile(CONFIG.VISITED_FILE, (body:gsub(probe .. "\n", "")))
            end
        end)
        print(("[ghost] server memory OK - %d server(s) remembered, will not revisit them")
            :format(before))
    else
        warn("[ghost] !! FILE IO NOT WORKING - visited servers are NOT being saved,")
        warn("[ghost] !! so hopping may land you back in servers you already checked.")
    end
    return works, before
end

local function hop(say)
    if stale() then return end
    clearWeld(true)
    say("hopping server...")
    local visited = loadVisited()
    visited[game.JobId] = true
    saveVisited(game.JobId)

    -- Walk the list from BOTH ends. The endpoint stops paginating well before
    -- it has shown you every server, so Desc alone only ever reaches one slice
    -- of them - Asc reaches a different one, roughly doubling what we can see.
    local pool, seen = {}, {}
    for _, order in ipairs({ "Desc", "Asc" }) do
        local cursor = ""
        for _ = 1, CONFIG.HOP_PAGES do
            local url = ("https://games.roblox.com/v1/games/%d/servers/Public?sortOrder=%s&limit=100&cursor=%s")
                :format(game.PlaceId, order, cursor)
            local body = httpGet(url)
            if not body then break end
            local ok, data = pcall(function() return HttpSvc:JSONDecode(body) end)
            if not ok or not data or not data.data then break end
            for _, srv in ipairs(data.data) do
                if type(srv.id) == "string" and srv.id ~= game.JobId
                   and not visited[srv.id] and not seen[srv.id]
                   and srv.playing and srv.maxPlayers and srv.playing < srv.maxPlayers then
                    seen[srv.id] = true
                    pool[#pool + 1] = srv
                end
            end
            cursor = data.nextPageCursor or ""
            if cursor == "" then break end
            task.wait(0.25)
        end
    end
    if #pool == 0 then
        -- The list ran dry. Rather than stall, forget the OLDEST half of the
        -- visited servers: those JobIds are the ones most likely to have been
        -- recycled by Roblox since, so they are effectively new servers now.
        -- The recent half stays remembered, so we do not immediately re-check
        -- what we just checked.
        -- Keep only the most recent few. The old version halved the FILE's
        -- lines, but each hop writes two of them and there are duplicates, so
        -- 90 remembered servers only dropped to 80 - not enough to free
        -- anything up. Dedupe first, then keep a small recent window.
        local kept = {}
        pcall(function()
            if not (isfile and readfile and writefile and isfile(CONFIG.VISITED_FILE)) then return end
            local order, seenId = {}, {}
            for id in tostring(readfile(CONFIG.VISITED_FILE)):gmatch("[^\r\n]+") do
                if id ~= "" and not seenId[id] then
                    seenId[id] = true
                    order[#order + 1] = id
                end
            end
            local from = math.max(1, #order - CONFIG.KEEP_RECENT + 1)
            for i = from, #order do kept[#kept + 1] = order[i] end
            writefile(CONFIG.VISITED_FILE, table.concat(kept, "\n") .. "\n")
        end)
        say(("server list ran dry - now remembering only the last %d, press CHECK")
            :format(#kept))
        print("[ghost] press HOP (or CHECK) again and it will have candidates.")
        return
    end
    local pick = pool[math.random(1, #pool)]
    saveVisited(pick.id)   -- mark it before we go, so a crash cannot lose it
    say(("hopping to a new server (%d unvisited candidates)"):format(#pool))
    pcall(function() TS:TeleportToPlaceInstance(game.PlaceId, pick.id, LP) end)
end

--==============================================================================
-- the pass
--==============================================================================
local busy = false

local function check(say)
    if busy then return end
    if stale() then
        print("[ghost] another copy of the script started - this one is stopping")
        return
    end
    busy = true
    clearWeld(true)

    local sites = ghillieSites()
    if #sites == 0 then
        say("no ghillie sites in this server - HOP")
        busy = false
        return
    end

    local veh, dist = nearestVehicle()
    if not veh then say(tostring(dist)) busy = false return end

    local home = myPos()
    local weld, partName = attach(veh)
    if not weld then say(tostring(partName)) busy = false return end
    say(("welded to %s (%s) - riding it to %d site(s)")
        :format(veh.Name, partName, #sites))

    local results, anyLoot = {}, nil
    local allEntries = {}   -- every crate read this pass, for the hop log
    local confirmed = 0     -- sites definitively read as EMPTY
    local unknown   = 0     -- sites we could not read at all

    for i, site in ipairs(sites) do
        local me = myPos() or home
        local dir = Vector3.new(1, 0, 0)
        local flat = Vector3.new(site.pos.X - me.X, 0, site.pos.Z - me.Z)
        if flat.Magnitude > 1 then dir = flat.Unit end
        local stop = site.pos - dir * CONFIG.APPROACH + Vector3.new(0, CONFIG.LIFT, 0)

        if not moveVehicle(veh, stop) then
            say("could not move the vehicle")
            unknown = unknown + 1
            break
        end
        say(("site %d/%d: %s - waiting for it to finish loading")
            :format(i, #sites, site.name))

        -- The crate EXISTING is not the same as the crate being loaded. Wait
        -- for three things: a minimum dwell, the ghillie crate to appear, and
        -- the site to stop gaining LootGroups - only then is it settled.
        local t0        = os.clock()
        local group     = nil
        local lastCount = -1
        local stableAt  = nil
        local empty, err
        local entries, readCount = {}, 0

        while os.clock() - t0 < CONFIG.HOLD_SECS do
            if stale() then
                clearWeld(true)
                print("[ghost] superseded by a newer copy - stopping mid-pass")
                busy = false
                return
            end
            moveVehicle(veh, stop)   -- keep holding position

            local g, count = ghillieGroup(site.pos)
            group = g or group

            if count ~= lastCount then
                lastCount = count
                stableAt = os.clock()      -- still growing, restart the timer
            end

            local dwelled  = os.clock() - t0 >= CONFIG.MIN_DWELL
            local settled  = stableAt and (os.clock() - stableAt >= CONFIG.STABLE_SECS)

            if group and dwelled and settled then
                -- read EVERY crate here, not just the ghillie one. a genuine
                -- read looks like a mix; a default looks like total agreement.
                for try = 1, CONFIG.READ_TRIES do
                    entries = siteGroups(site.pos)
                    readCount = readAll(entries)
                    -- ANY full ghillie crate in range wins. The old version
                    -- assigned in a loop, so the LAST crate read decided it -
                    -- a full one could be overwritten by an empty one and the
                    -- script would hop straight past the thing we are hunting.
                    local sawGhillie, anyFull, allRead = false, false, true
                    for _, e in ipairs(entries) do
                        if e.ghillie then
                            sawGhillie = true
                            if e.empty == false then anyFull = true
                            elseif e.empty == nil then allRead = false end
                        end
                    end
                    if anyFull then
                        empty = false                    -- loot: stop, never hop
                    elseif sawGhillie and allRead then
                        empty = true                     -- every one confirmed empty
                    end
                    if empty ~= nil then break end
                    say(("site %d: state not ready, re-reading (%d/%d)")
                        :format(i, try, CONFIG.READ_TRIES))
                    task.wait(CONFIG.POLL)
                    moveVehicle(veh, stop)
                end
                if empty ~= nil then break end
            end

            task.wait(CONFIG.POLL)
        end

        if not group then
            unknown = unknown + 1
            results[#results + 1] = ("%s: crate never streamed"):format(site.name)
            say(("site %d: crate never streamed in %ds"):format(i, CONFIG.HOLD_SECS))
        else
            -- the census: this is what tells us whether to believe any of it
            local nEmpty, nLoot = 0, 0
            for _, e in ipairs(entries) do
                if e.empty == true then nEmpty = nEmpty + 1
                elseif e.empty == false then nLoot = nLoot + 1 end
            end
            for _, e in ipairs(entries) do allEntries[#allEntries + 1] = e end
            print(("[census] %s - %d crates, %d read: %d empty, %d with loot")
                :format(site.name, #entries, readCount, nEmpty, nLoot))
            for _, e in ipairs(entries) do
                print(("    %-34s %-9s %s")
                    :format(e.label,
                            e.empty == nil and "UNREAD" or (e.empty and "empty" or "HAS LOOT"),
                            e.ghillie and (e.strict and "<-- GHILLIE"
                                                    or "<-- GHILLIE (name only)") or ""))
            end
            results[#results + 1] = ("%s census: %d empty / %d loot / %d unread")
                :format(site.name, nEmpty, nLoot, #entries - readCount)

            say(("site %d: %d crates, %d empty, %d with loot")
                :format(i, #entries, nEmpty, nLoot))
            if empty == nil then
                unknown = unknown + 1
                results[#results + 1] = ("%s: unreadable (%s)")
                    :format(site.name, tostring(err or "timed out"))
                say(("site %d: could not read state (%s)")
                    :format(i, tostring(err or "timed out")))
            elseif empty then
                confirmed = confirmed + 1
                results[#results + 1] = ("%s: EMPTY"):format(site.name)
                say(("site %d (%s): EMPTY"):format(i, site.name))
            else
                anyLoot = site
                results[#results + 1] = ("%s: HAS LOOT"):format(site.name)
                say(("site %d (%s): *** HAS LOOT ***"):format(i, site.name))
                break
            end
        end
    end

    clearWeld(true)
    task.wait(0.4)

    -- Log every server we check, so the question "which servers are worth
    -- joining?" gets answered by data instead of by guesswork. The API cannot
    -- tell us a server's AGE, so the theory that fresh servers are better is
    -- untestable directly - but the share of crates already looted is a decent
    -- stand-in for how picked-over a server is, and player count is free.
    pcall(function()
        if not appendfile then return end
        local nE, nL = 0, 0
        for _, e in ipairs(allEntries) do
            if e.empty == true then nE = nE + 1
            elseif e.empty == false then nL = nL + 1 end
        end
        local ratio = (nE + nL) > 0 and (nL / (nE + nL)) or 0
        appendfile("ar2_hop_log.csv", ("%s,%s,%d,%d,%d,%d,%.3f,%s\n"):format(
            os.date("%Y-%m-%d %H:%M:%S"),
            game.JobId,
            #Players:GetPlayers(),
            Players.MaxPlayers,
            nL, nE, ratio,
            anyLoot and "GHILLIE" or "none"))
    end)

    print("---------------- GHOST CHECK ----------------")
    for _, line in ipairs(results) do print("  " .. line) end
    print("---------------------------------------------")

    if anyLoot then
        say(("*** GHILLIE AT %s - GO GET IT ***"):format(anyLoot.name))
        print(("[ghost] target: %s at %d, %d, %d")
            :format(anyLoot.name, anyLoot.pos.X, anyLoot.pos.Y, anyLoot.pos.Z))
        print("[ghost] that crate's loot table has exactly ONE entry, the ghillie,")
        print("[ghost] so it is not a maybe - drive there and open it.")
        print("[ghost] NOT hopping. Press HOP yourself if you want to leave it.")
    else
        -- Only hop when EVERY site was definitively read as empty. An
        -- unreadable site is not an empty one, and hopping past it could throw
        -- away the very crate we are hunting.
        if unknown > 0 then
            say(("%d site(s) could not be read - NOT hopping, press CHECK again")
                :format(unknown))
            print("[ghost] refusing to hop: an unread site might have the ghillie.")
            print("[ghost] press CHECK to retry this server, or HOP to leave anyway.")
        elseif confirmed < CONFIG.EXPECTED_SITES then
            -- You said there are always two crash sites per server. Finding
            -- fewer means one was not listed, or two were folded together, so
            -- hopping now would leave a site unchecked.
            say(("only %d site(s) checked, expected %d - NOT hopping, press CHECK again")
                :format(confirmed, CONFIG.EXPECTED_SITES))
            print("[ghost] fewer sites than expected. press CHECK to retry, or HOP to")
            print("[ghost] leave anyway. Lower EXPECTED_SITES if this server really has one.")
        else
            say(("all %d ghillie crate(s) confirmed empty - hopping"):format(confirmed))
            if CONFIG.AUTO_HOP then hop(say) end
        end
    end
    busy = false
end

--==============================================================================
-- GUI
--==============================================================================
do
    local g = (getgenv and getgenv()) or _G
    if g.AR2_GHOST_GUI then pcall(function() g.AR2_GHOST_GUI:Destroy() end) end
end

local gui = Instance.new("ScreenGui")
gui.Name = "AR2GhostCheck"
gui.ResetOnSpawn = false
gui.Parent = (gethui and gethui()) or game:GetService("CoreGui")
do local g = (getgenv and getgenv()) or _G; g.AR2_GHOST_GUI = gui end

local f = Instance.new("Frame")
f.Size = UDim2.new(0, 290, 0, 108)
f.Position = UDim2.new(0, 30, 0, 130)
f.BackgroundColor3 = Color3.fromRGB(18, 18, 20)
f.BorderSizePixel = 0
f.Active, f.Draggable = true, true
f.Parent = gui
Instance.new("UICorner", f).CornerRadius = UDim.new(0, 8)
local stroke = Instance.new("UIStroke", f)
stroke.Color = Color3.fromRGB(60, 60, 66)

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -20, 0, 20)
title.Position = UDim2.new(0, 10, 0, 6)
title.BackgroundTransparency = 1
title.Font, title.TextSize = Enum.Font.GothamBold, 13
title.TextColor3 = Color3.fromRGB(180, 140, 255)
title.TextXAlignment = Enum.TextXAlignment.Left
title.Text = "GHOST CHECK"
title.Parent = f

local status = Instance.new("TextLabel")
status.Size = UDim2.new(1, -20, 0, 44)
status.Position = UDim2.new(0, 10, 0, 26)
status.BackgroundTransparency = 1
status.Font, status.TextSize = Enum.Font.Gotham, 11
status.TextColor3 = Color3.fromRGB(160, 160, 168)
status.TextXAlignment = Enum.TextXAlignment.Left
status.TextYAlignment = Enum.TextYAlignment.Top
status.TextWrapped = true
status.Text = "CHECK reads both crash sites without you going there."
status.Parent = f

local function say(s) status.Text = s print("[ghost] " .. s) end

local function mk(txt, x, w, col)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(0, w, 0, 28)
    b.Position = UDim2.new(0, x, 1, -34)
    b.BackgroundColor3 = col
    b.BorderSizePixel = 0
    b.Font, b.TextSize = Enum.Font.GothamBold, 12
    b.TextColor3 = Color3.fromRGB(15, 15, 18)
    b.Text = txt
    b.Parent = f
    Instance.new("UICorner", b).CornerRadius = UDim.new(0, 6)
    return b
end

mk("CHECK", 10, 84, Color3.fromRGB(180, 140, 255)).MouseButton1Click:Connect(function()
    task.spawn(check, say)
end)
mk("HOP", 100, 64, Color3.fromRGB(255, 200, 40)).MouseButton1Click:Connect(function()
    task.spawn(hop, say)
end)
mk("CLEAR", 170, 64, Color3.fromRGB(140, 240, 160)).MouseButton1Click:Connect(function()
    clearWeld(false)
    say("unwelded - you will have snapped home")
end)
mk("X", 240, 40, Color3.fromRGB(200, 90, 90)).MouseButton1Click:Connect(function()
    clearWeld(true)
    gui:Destroy()
end)

LP.CharacterAdded:Connect(function() task.wait(0.5) clearWeld(true) end)

-- carry the loop across a hop if the executor supports it. this needs the file
-- saved in the executor's workspace folder under this name.
local function queueNext()
    local q = queue_on_teleport or queueonteleport or (syn and syn.queue_on_teleport)
    if type(q) ~= "function" then return false end
    local ok = pcall(q, [[
        if game.PlaceId == 863266079 and isfile and isfile("ar2_ghost_check.lua") then
            loadstring(readfile("ar2_ghost_check.lua"))()
        end
    ]])
    return ok
end

checkMemory()
say("ready - CHECK reads both sites, no travel needed")
if CONFIG.AUTO_START then
    task.spawn(function()
        task.wait(CONFIG.START_DELAY)
        if CONFIG.QUEUE_ON_HOP and queueNext() then
            print("[ghost] queued to resume after the next hop")
        end
        check(say)
    end)
end
