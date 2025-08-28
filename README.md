repeat task.wait() until workspace:FindFirstChild("Plots")
local HttpService = game:GetService("HttpService")

local UnderTen = ""
local OverTen = ""
local embedColor = 3447003
local placeId = game.PlaceId

local AllIDs = {}
local foundAnything = ""
local actualHour = os.date("!*t").hour
local Deleted = false

function TPReturner()
   local Site;
   if foundAnything == "" then
       Site = game.HttpService:JSONDecode(game:HttpGet('https://games.roblox.com/v1/games/' .. placeId .. '/servers/Public?sortOrder=Asc&limit=100'))
   else
       Site = game.HttpService:JSONDecode(game:HttpGet('https://games.roblox.com/v1/games/' .. placeId .. '/servers/Public?sortOrder=Asc&limit=100&cursor=' .. foundAnything))
   end
   local ID = ""
   if Site.nextPageCursor and Site.nextPageCursor ~= "null" and Site.nextPageCursor ~= nil then
       foundAnything = Site.nextPageCursor
   end
   local num = 0;
   for i,v in pairs(Site.data) do
       local Possible = true
       ID = tostring(v.id)
       if tonumber(v.maxPlayers) > tonumber(v.playing) then
           for _,Existing in pairs(AllIDs) do
               if num ~= 0 then
                   if ID == tostring(Existing) then
                       Possible = false
                   end
               else
                   if tonumber(actualHour) ~= tonumber(Existing) then
                       local delFile = pcall(function()
                           delfile("NotSameServers.json")
                           AllIDs = {}
                           table.insert(AllIDs, actualHour)
                       end)
                   end
               end
               num = num + 1
           end
           if Possible == true then
               table.insert(AllIDs, ID)
               task.wait()
               pcall(function()
                   writefile("NotSameServers.json", game:GetService('HttpService'):JSONEncode(AllIDs))
                   wait()
                   game:GetService("TeleportService"):TeleportToPlaceInstance(placeId, ID, game.Players.LocalPlayer)
               end)
               task.wait(0.1)
           end
       end
   end
end
 
function Teleport()
   while task.wait() do
       pcall(function()
           TPReturner()
           if foundAnything ~= "" then
               TPReturner()
           end
       end)
   end
end

local req = (syn and syn.request) or (http and http.request) or (http_request) or (fluxus and fluxus.request) or request

local function sendWebhook(url, petFound, moneyPerSec, tag, mutation, jobId, pingEveryone)
    local data = {
        ["username"] = "Ken Hub",
        ["embeds"] = {{
            ["title"] = "Ken Hub",
            ["description"] = petFound,
            ["color"] = embedColor,
            ["fields"] = {
                {["name"] = "💸 Money per Sec", ["value"] = moneyPerSec, ["inline"] = true},
                {["name"] = "🏷️ Tag", ["value"] = tag, ["inline"] = true},
                {["name"] = "🧬 Mutation", ["value"] = mutation, ["inline"] = true},
                {["name"] = "Join Link", ["value"] = "[Join Here](https://www.roblox.com/games/start?placeId="..placeId.."&launchData="..jobId..")", ["inline"] = false},
                {["name"] = "🆔 Job ID (Mobile)", ["value"] = "``"..jobId.."``", ["inline"] = false},
                {["name"] = "🆔 Job ID (PC)", ["value"] = "```"..jobId.."```", ["inline"] = false},
                {["name"] = "Join Server",
                 ["value"] = string.format("```lua\ngame:GetService('TeleportService'):TeleportToPlaceInstance(%d, '%s', game.Players.LocalPlayer)\n```", placeId, jobId),
                 ["inline"] = false
                }
            }
        }}
    }
    
    if pingEveryone then
        data["content"] = "@everyone"
    end
    
    local jsonData = HttpService:JSONEncode(data)
    
    req({
        Url = url,
        Method = "POST",
        Headers = {["Content-Type"] = "application/json"},
        Body = jsonData
    })
end

local function convertTextToNumber(text)
    if not text:find("/s") then
        return 0
    end
    
    text = text:gsub("%$", ""):gsub("/s", ""):gsub("%s+", "")
    
    local multiplier = 1
    
    if text:upper():find("K$") then
        multiplier = 1e3
        text = text:gsub("[Kk]$", "")
    elseif text:upper():find("M$") then
        multiplier = 1e6
        text = text:gsub("[Mm]$", "")
    elseif text:upper():find("B$") then
        multiplier = 1e9
        text = text:gsub("[Bb]$", "")
    elseif text:upper():find("T$") then
        multiplier = 1e12
        text = text:gsub("[Tt]$", "")
    end
    
    local num = tonumber(text) or 0
    return num * multiplier
end

local webhooksToSend = {}

for _, v in pairs(game:GetService("Workspace"):GetDescendants()) do
    if v:IsA("TextLabel") and v.Name == "Generation" then
        local originalText = v.Text
        local value = convertTextToNumber(originalText)
        
        if value >= 1000000 then
            local success, result = pcall(function()
                local petFound = v.Parent.DisplayName.Text
                local moneyPerSec = v.Text
                local tag = v.Parent.Rarity.Text
                local jobId = game.JobId
                
                local mutation = "None"
                local mutationTag = v.Parent:FindFirstChild("Mutation")
                if mutationTag and mutationTag.Visible then
                    mutation = mutationTag.Text
                end
                
                return petFound, moneyPerSec, tag, mutation, jobId
            end)
            
            if success then
                local petFound, moneyPerSec, tag, mutation, jobId = result, v.Text, v.Parent.Rarity.Text, "None", game.JobId
                
                local mutationTag = v.Parent:FindFirstChild("Mutation")
                if mutationTag and mutationTag.Visible then
                    mutation = mutationTag.Text
                end
                
                local webhookUrl, shouldPing
                if petFound:lower() == "lucky block" then
                    if tag:lower() == "secret" then
                        webhookUrl = OverTen
                        shouldPing = false
                    else
                        webhookUrl = UnderTen
                        shouldPing = false
                    end
                else
                    webhookUrl = value >= 10000000 and OverTen or UnderTen
                    shouldPing = value >= 10000000
                end
                
                table.insert(webhooksToSend, {
                    url = webhookUrl,
                    petFound = petFound,
                    moneyPerSec = moneyPerSec,
                    tag = tag,
                    mutation = mutation,
                    jobId = jobId,
                    pingEveryone = shouldPing
                })
            else
                warn("Error accessing pet data:", result)
            end
        end
    end
end

for _, webhookData in pairs(webhooksToSend) do
    sendWebhook(webhookData.url, webhookData.petFound, webhookData.moneyPerSec, webhookData.tag, webhookData.mutation, webhookData.jobId, webhookData.pingEveryone)
    task.wait(0.1)
end

if #webhooksToSend > 0 then
    print("Sent", #webhooksToSend, "webhooks. Starting teleport...")
    task.wait(0.5)
end

Teleport()