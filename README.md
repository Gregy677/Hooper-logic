--// Server Hopper Script (Standalone)
local HttpService = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer

local PlaceID = game.PlaceId
local Cursor = nil
local SeenServers = {}
local MaxAttempts = 10   -- How many pages to check per hop
local HopDelay = 3.2     -- Seconds between hop attempts

-- Load previously seen servers from file
local function loadSeenServers()
    pcall(function()
        local data = readfile("SeenServers.json")
        SeenServers = HttpService:JSONDecode(data)
    end)
end

-- Save seen servers to file
local function saveSeenServers()
    pcall(function()
        writefile("SeenServers.json", HttpService:JSONEncode(SeenServers))
    end)
end

-- Find a new server to hop to
local function findNewServer()
    local attempts = 0
    while attempts < MaxAttempts do
        local url = ("https://games.roblox.com/v1/games/%d/servers/Public?sortOrder=Asc&limit=100%s"):format(
            PlaceID,
            Cursor and ("&cursor=" .. Cursor) or ""
        )

        local success, result = pcall(function()
            return HttpService:JSONDecode(game:HttpGet(url))
        end)

        if success and result and result.data then
            for _, server in ipairs(result.data) do
                if server.playing < server.maxPlayers and not SeenServers[server.id] then
                    SeenServers[server.id] = true
                    saveSeenServers()
                    return server.id
                end
            end
            Cursor = result.nextPageCursor
            if not Cursor then break end
        else
            break
        end
        attempts += 1
    end
    return nil
end

-- Actually hop to a server
local function hopServer()
    local newServerID = findNewServer()
    if newServerID then
        TeleportService:TeleportToPlaceInstance(PlaceID, newServerID, LocalPlayer)
    else
        warn("⚠ No new servers found!")
    end
end

-- Start hopper loop
loadSeenServers()
task.spawn(function()
    while true do
        task.wait(HopDelay)
        hopServer()
    end
end)
