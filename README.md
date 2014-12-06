#pragma dynamic 20000

#include <a_samp>
#include <a_npc>
#define FIXES_Single 1
#define MODE_NAME
#include <OnlyRP\orp_params>



#define DIALOG_STATS                                                            1
#define DIALOG_CREATION_VEH                                                     2
#define DIALOG_AIDE_MAIN                                                        3
#define DIALOG_AIDE_GENERAL                                                     4
#define DIALOG_AIDE_COMPTE                                                      5

/*#define DIALOG_OBJETS_CREER                                                     900
#define DIALOG_OBJETS_LISTE                                                     910*/


/*----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------*/
#define MAX_BOOL_VAR                                                            (100)             // Max de variable dans l'enumeration

#define State_Get(%0,%1)            ((%0) & (%1))        // Retourne 0 ou 1
#define State_On(%0,%1)             ((%0) |= (%1))      // Met sur 1
#define State_Off(%0,%1)            ((%0) &= ~(%1))     // Met sur off
#define State_Toggle(%0,%1)         ((%0) ^= (%1))      // Toggle -> stock le contraire de la valeur d'origine

enum StateBool:(<<= 1) {                                                                         // Variable trés legere, booléen

   PLAYER_CONNECTED = 1,     // CONNECT = 1 != DISCONNECT
   PLAYER_MUTED,             // MUTE = 1 != UNMUTE
   PLAYER_FREEZE,            // FREEZE = 1 != UNFREEZE
   PLAYER_ADMIN_DUTY,        // DUTY = 1 != UNDUTY
   PLAYER_COMPTEUR_SHOW,
   PLAYER_CEINTURE_MISE

};
new StateBool:VarState[MAX_PLAYERS];
/*----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------*/


/*----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------*/
#include <OnlyRP\orp_enum>
/*----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------*/


new terrain[MAX_PLAYERS][30];
new toit[MAX_PLAYERS][12];
new mur[MAX_PLAYERS][25];

// Player connexion
new launcher_ip[MAX_PLAYERS][16];
new launcher_date[MAX_PLAYERS][16];
new launcher_heure[MAX_PLAYERS][8];

// Véhicule
new StartEngine[MAX_PLAYERS];


// Edit object
#define EDIT_ITEM_OBJECT_REGZ                                                   1
#define EDIT_ITEM_OBJECT_ROTATION                                               2
#define EDIT_ITEM_OBJECT_HAND_LR                                                3
#define EDIT_ITEM_OBJECT_HAND_TOTAL                                             4
new IPO[MAX_PLAYERS];   // ItemPlayerObject
new Float:posZinit[MAX_PLAYERS];


forward OnPlayerDataLoad(playerid);
//forward OnObjectsDataLoad();
//forward KickEx(playerid, auteur[], raison[], silencieux);
forward TimerKick(playerid);

/*----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------*/
#include <OnlyRP\orp_textdraw>
#include <OnlyRP\orp_objects>
#include <OnlyRP\orp_timers>
#include <OnlyRP\orp_function>
#include <OnlyRP\orp_vehicle>
#include <OnlyRP\orp_admin>
#include <OnlyRP\orp_dialog>
#include <OnlyRP\orp_trousseau>
/*----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------*/

main()
{
    print("\n----------------------------------");
    print(SERVER_NAME);
    print("----------------------------------\n");
}

public OnGameModeInit()
{
    #include <i_altcmd>
    ConnectMySQL();

    //EnableZoneNames(0);
    new phrase[STRING_0];
    format(phrase,sizeof(phrase),"%s %.1f",SERVER_NAME, SERVER_VERSION);
    SetGameModeText(phrase);
    AddPlayerClass(0, 1958.33, 1343.12, 15.36, 269.15, 26, 36, 28, 150, 0, 0);

    // Chargements
    print("Chargement des fichiers ...");
    LoadModelVehicle();
    
    // Textdraw
    CompteurVeh_TD();

    // NPC
	ConnectNPC("Arthur", "Banque");
    return 1;
}

public OnGameModeExit()
{
    mysql_close();
    return 1;
}
public OnPlayerRequestClass(playerid, classid)
{
	if(IsPlayerNPC(playerid))
	{
		SpawnPlayer(playerid);
		SetPlayerPos(playerid, 0, 0, 0);
		return 1;
	}
    SetPlayerSpawn(playerid);
    return 1;
}

public OnPlayerConnect(playerid)
{
    #include <OnlyRP\MAPPING\REMOVE>
    #include <OnlyRP\MAPPING\TERRAINS>
    #include <OnlyRP\MAPPING\TOITS>

    State_On(VarState[playerid], PLAYER_CONNECTED);
    State_Off(VarState[playerid], PLAYER_MUTED);
    State_Off(VarState[playerid], PLAYER_ADMIN_DUTY);
    State_Off(VarState[playerid], PLAYER_FREEZE);
    State_Off(VarState[playerid], PLAYER_COMPTEUR_SHOW);
    State_Off(VarState[playerid], PLAYER_CEINTURE_MISE);
    
    
    new requete[STRING_3];
    format(requete,sizeof(requete),"SELECT * FROM `utilisateurs` WHERE pseudo='%s' LIMIT 1", GetPName(playerid));
    mysql_tquery(mysql_handle, requete);
    
    new rows, fields;
    cache_get_data(rows, fields, mysql_handle);
    if(rows)
    {
        cache_get_field_content(rows, "L_Date", launcher_date[playerid], bug_mysql_handle, 16);
        cache_get_field_content(rows, "L_Heure", launcher_date[playerid], bug_mysql_handle, 8);
        cache_get_field_content(rows, "L_IP", launcher_date[playerid], bug_mysql_handle, 16);
	}
	else
	{
	    SendMessage(playerid,MSG_INFO,"Vous n'avez pas de compte.");
        KickEx(playerid,"[SERVEUR]","Pas de compte",0);
        return 1;
	}
	
	new dateformat[16], heureformat[8], heure, minute, second, jour, mois, annee,  l_heure[4], l_minute[4], l_second[4];
	getdate(annee, mois, jour);
	gettime(heure, minute, second)
	
	new oldpos, pos;
    pos = strfind(launcher_heure[playerid], ":", true, 0);
	strmid(l_heure, launcher_heure[playerid], 0, pos-1);
	
	oldpos=pos;
	pos=strfind(launcher_heure[playerid], ":", true, oldpos+1);
	strmid(l_minute, launcher_heure[playerid], oldpos+1, pos-1);
	
	strmid(l_second, launcher_heure[playerid], pos+1, strlen(launcher_heure[playerid]));
	
	if(strcmp(PInfo[playerid][token],GetPIp(playerid)) != 0)
    {
        SendMessage(playerid,MSG_ERREUR,"Token invalide. Veuillez vous reconnecter depuis l'application. Si le problème persiste, contactez l'administration.");
        KickEx(playerid,"[SERVEUR]","Token Invalide",0);
    }
	
    /*if(!IsPlayerNPC(playerid))
	{
	    new requete[STRING_3];
	    format(requete,sizeof(requete),"SELECT * FROM `personnages` WHERE nom='%s' LIMIT 1",GetPName(playerid));
	    mysql_function_query(mysql_handle, requete, true, "OnPlayerDataLoad", "i", playerid);
	    LoadPlayerTextures(playerid);
	}*/
	
	// SAMP +
	/*ToggleHUDComponentForPlayer(playerid, HUD_COMPONENT_WEAPON, false);
	ToggleHUDComponentForPlayer(playerid, HUD_COMPONENT_AMMO, false);
	ToggleHUDComponentForPlayer(playerid, HUD_COMPONENT_MINIMAP, false);
	ToggleHUDComponentForPlayer(playerid, HUD_COMPONENT_MONEY, false);*/
    return 1;
}
public OnPlayerDisconnect(playerid, reason)
{
    State_Off(VarState[playerid], PLAYER_CONNECTED);
	KillTimerForPlayer(playerid);
	if(GetPlayerState(playerid) == PLAYER_STATE_DRIVER)
	{
		new vehicleid = GetPlayerVehicleID(playerid);
		SaveVehiclePos(vehicleid);
		SaveVehicle(vehicleid);
	}
    return 1;
}

public OnPlayerSpawn(playerid)
{
    ClearChat(playerid,30);
    return 1;
}

public OnPlayerDeath(playerid, killerid, reason)
{
    return 1;
}

public OnVehicleSpawn(vehicleid)
{
    return 1;
}

public OnVehicleDeath(vehicleid, killerid)
{
    return 1;
}

public OnPlayerText(playerid, text[])
{
    if(!State_Get(VarState[playerid], PLAYER_CONNECTED))
        {SendMessage(playerid,MSG_ERREUR,Msg_Serveur[NOT_CONNECTED]);
         return 0;
    }
    if(State_Get(VarState[playerid], PLAYER_MUTED))
        {SendMessage(playerid,MSG_INFO,Msg_Serveur[MUTED]);
         return 0;
    }
    TchatMsg(playerid, text, LOCAL_TCHAT_IC);
    return 0;
}

/*******************************************************/
#include <OnlyRP\orp_commands>
/******************************************************/

public OnPlayerClickMap(playerid, Float:fX, Float:fY, Float:fZ)
{
	if(State_Get(VarState[playerid], PLAYER_ADMIN_DUTY))
	{
        if(GetPlayerState(playerid) == PLAYER_STATE_DRIVER)
	   	{
	   	    new vehicleid = GetPlayerVehicleID(playerid);
            SetVehiclePos(vehicleid, fX, fY, fZ);
			PutPlayerInVehicle(playerid, vehicleid, 0);
	   	}
	   	else
	   	{
		    SetPlayerPos(playerid,fX,fY,fZ);
		}
	}
	return 1;
}
public OnPlayerEnterVehicle(playerid, vehicleid, ispassenger)
{
    return 1;
}

public OnPlayerExitVehicle(playerid, vehicleid)
{
	if(VInfo[vehicleid][v_key] == 1)
	{
        VInfo[vehicleid][v_key] = 0;
        return 1;
	}
	SaveVehiclePos(vehicleid);
	SaveVehicle(vehicleid);
    return 1;
}

public OnPlayerStateChange(playerid, newstate, oldstate)
{
    return 1;
}

public OnPlayerEnterCheckpoint(playerid)
{
    return 1;
}

public OnPlayerLeaveCheckpoint(playerid)
{
    return 1;
}

public OnPlayerEnterRaceCheckpoint(playerid)
{
    return 1;
}

public OnPlayerLeaveRaceCheckpoint(playerid)
{
    return 1;
}

public OnRconCommand(cmd[])
{
    return 1;
}

public OnPlayerRequestSpawn(playerid)
{

    return 1;
}

public OnObjectMoved(objectid)
{
    return 1;
}

public OnPlayerObjectMoved(playerid, objectid)
{
    return 1;
}

public OnPlayerPickUpPickup(playerid, pickupid)
{
    return 1;
}

public OnVehicleMod(playerid, vehicleid, componentid)
{
    return 1;
}

public OnVehiclePaintjob(playerid, vehicleid, paintjobid)
{
    return 1;
}

public OnVehicleRespray(playerid, vehicleid, color1, color2)
{
    return 1;
}

public OnPlayerSelectedMenuRow(playerid, row)
{
    return 1;
}

public OnPlayerExitedMenu(playerid)
{
    return 1;
}

public OnPlayerInteriorChange(playerid, newinteriorid, oldinteriorid)
{
    return 1;
}

public OnPlayerKeyStateChange(playerid, newkeys, oldkeys)
{
	if(PRESSED(KEY_LOOK_LEFT))
	{
		if(GetPlayerState(playerid) == PLAYER_STATE_DRIVER)
		{
			new vehicleid = GetPlayerVehicleID(playerid);
			if(VehicleContactIsStart(vehicleid))
			{
                ToggleCligno(vehicleid, 1);
			}
		}
	}
	if(PRESSED(KEY_LOOK_RIGHT))
	{
		if(GetPlayerState(playerid) == PLAYER_STATE_DRIVER)
		{
			new vehicleid = GetPlayerVehicleID(playerid);
			if(VehicleContactIsStart(vehicleid))
			{
                ToggleCligno(vehicleid, 0);
			}
		}
	}
    if(PRESSED(KEY_WALK))
    {
        new player = GetPlayerTargetPlayer(playerid);
        if(player != INVALID_PLAYER_ID)
        {
			new Float:pos[3];
			GetPlayerPos(playerid, pos[0], pos[1], pos[2]);
			if(GetPlayerDistanceFromPoint(playerid, pos[0], pos[1], pos[2]) <= 1.0)
			{
		        Dialog_ShowMenuInteraction(playerid, player);
			}
        }
    }
    if(PRESSED(KEY_FIRE))
    {
        if(GetPlayerState(playerid) == PLAYER_STATE_DRIVER)
        {
            new vehicleid = GetPlayerVehicleID(playerid);
            if(VehicleHasEngine(vehicleid) == -1)return 1;                      // Demarrage Vehicule
            if(State_Get(VarState[playerid], PLAYER_ADMIN_DUTY) || VInfo[vehicleid][v_key] == 1)
            {
				if(GetVehicleBatterie(vehicleid) <= 0) return SM_(playerid, MSG_INFO, "Ce véhicule n'a plus de batterie.");
                if(VehicleContactIsStart(vehicleid))
                {
                    new key, a, b;
                    GetPlayerKeys(playerid, key, a ,b);
                    if(VehicleEngineIsStart(vehicleid) || StartEngine[playerid] > 0) return 1;
                    if(key == KEY_SPRINT) return SM_(playerid, MSG_INFO,"Le démarrage du véhicule a échoué.");
                    StartEngine[playerid]= vehicleid;
                    SetVehicleBatterie(vehicleid, GetVehicleBatterie(vehicleid)-BATTERIE_DEMARRAGE);
                    SM_(playerid, MSG_INFO, "Le véhicule démarre ...");
                    VInfo[vehicleid][v_batteries] -= BATTERIE_DEMARRAGE;
                    if(GetVehicleEssence(vehicleid) <= 0.0 || GetVehicleHuile(vehicleid) <= 0.0) return SM_(playerid, MSG_INFO,"Le démarrage du véhicule a échoué.");
                    SetTimerEx("pTimerEngine", 4000, false, "ii", playerid, vehicleid);
                }
                else{
                    StartVehicleContact(vehicleid);
                    SM_(playerid, MSG_INFO,"Vous venez de mettre le contact du véhicule.");
                }
            }
        }
    }
    if(RELEASED(KEY_FIRE))
    {
        if(StartEngine[playerid] > 0)                                           // Demarrage Vehicule
        {
            StartEngine[playerid] = 0;
            SM_(playerid, MSG_INFO,"Le démarrage du véhicule a échoué.");
        }
    }
    if((newkeys & (KEY_FIRE | KEY_ACTION)) == (KEY_FIRE | KEY_ACTION) && (oldkeys & (KEY_FIRE | KEY_ACTION)) != (KEY_FIRE | KEY_ACTION))
    {
        if(GetPlayerState(playerid) == PLAYER_STATE_DRIVER)
        {
            new vehicleid = GetPlayerVehicleID(playerid);
            if(VehicleHasEngine(vehicleid) == -1)return 1;                      // Couper moteur
            if(VehicleEngineIsStart(vehicleid)){
                StopVehicleEngine(vehicleid);
                SM_(playerid, MSG_INFO, "Vous venez de couper le moteur du véhicule.");
            }
        }
    }
    if(PRESSED(KEY_SPRINT))
    {
        if(GetPlayerState(playerid) == PLAYER_STATE_DRIVER)
        {
            if(StartEngine[playerid] > 0){                                      // Noyer Moteur
                StartEngine[playerid] = 0;
                SM_(playerid, MSG_INFO,"Le démarrage du véhicule a échoué.");
            }
        }
    }
    return 1;
}

public OnRconLoginAttempt(ip[], password[], success)
{
    return 1;
}

public OnPlayerUpdate(playerid)
{
	if(State_Get(VarState[playerid], PLAYER_CONNECTED))
	{
        UpdateVitesse(playerid);
	}
    return 1;
}

public OnPlayerStreamIn(playerid, forplayerid)
{
    return 1;
}

public OnPlayerStreamOut(playerid, forplayerid)
{
    return 1;
}

/*public OnVehicleStreamIn(vehicleid, forplayerid)
{
    if(VInfo[vehicleid][v_portes] == 1) SetVehicleParamsForPlayer(vehicleid,forplayerid,0,1);
    return 1;
}*/

public OnVehicleStreamOut(vehicleid, forplayerid)
{
    return 1;
}

public OnPlayerEditObject(playerid, playerobject, objectid, response, Float:fX, Float:fY, Float:fZ, Float:fRotX, Float:fRotY, Float:fRotZ)
{
    if(playerobject)
    {
		if(response ==  EDIT_RESPONSE_FINAL)
		{
			if(IPO[playerid] == EDIT_ITEM_OBJECT_REGZ)
			{
	            OMInfo[idstock[playerid]][om_sol_reglagez] = posZinit[playerid] - fZ;
	            DestroyPlayerObject(playerid, ObjectCreation[playerid]);
	            Dialog_ShowParamsList(playerid, idstock[playerid], OBJECT_ITEM_PARAMS);
	            return 1;
			}
			else if(IPO[playerid] == EDIT_ITEM_OBJECT_ROTATION)
			{
	            OMInfo[idstock[playerid]][om_sol_rotx] = fRotX;
	            OMInfo[idstock[playerid]][om_sol_roty] = fRotY;
	            OMInfo[idstock[playerid]][om_sol_rotz] = fRotZ;
	            DestroyPlayerObject(playerid, ObjectCreation[playerid]);
	            Dialog_ShowParamsList(playerid, idstock[playerid], OBJECT_ITEM_PARAMS);
	            return 1;
			}
		}
		else if(response == EDIT_RESPONSE_CANCEL)
		{
		    Dialog_ShowParamsList(playerid, idstock[playerid], OBJECT_ITEM_PARAMS);
		    DestroyPlayerObject(playerid, ObjectCreation[playerid]);
		    return 1;
		}
    }
    return 1;
}

public OnPlayerEditAttachedObject(playerid, response, index, modelid, boneid, Float:fOffsetX, Float:fOffsetY, Float:fOffsetZ, Float:fRotX, Float:fRotY, Float:fRotZ, Float:fScaleX, Float:fScaleY, Float:fScaleZ)
{
    if(response)
    {
        if(IPO[playerid] > 0)
        {
			new poshand = (boneid == 5 && IPO[playerid] == EDIT_ITEM_OBJECT_HAND_LR) ? HAND_LEFT : (boneid == 6) ? HAND_RIGHT : HAND_TOTAL;
			OPosHand[idstock[playerid]][o_hand_x][poshand] = fOffsetX;
			OPosHand[idstock[playerid]][o_hand_y][poshand] = fOffsetY;
			OPosHand[idstock[playerid]][o_hand_z][poshand] = fOffsetZ;
			OPosHand[idstock[playerid]][o_hand_rx][poshand] = fRotX;
			OPosHand[idstock[playerid]][o_hand_ry][poshand] = fRotY;
			OPosHand[idstock[playerid]][o_hand_rz][poshand] = fRotZ;
			OPosHand[idstock[playerid]][o_hand_tx][poshand] = fScaleX;
			OPosHand[idstock[playerid]][o_hand_ty][poshand] = fScaleY;
			OPosHand[idstock[playerid]][o_hand_tz][poshand] = fScaleZ;
			RemovePlayerAttachedObject(playerid, 0);
		    SetPlayerSpecialAction(playerid,SPECIAL_ACTION_NONE);
            SPD(playerid, MENU_CREATE_MODEL_OBJECT, DIALOG_STYLE_LIST, "Main Attach", (OMInfo[idstock[playerid]][om_position] == HAND_POSITION_LR) ? ("Main Gauche\nMain Droite") : ("Main Gauche/Droite"), "Continuer", "Retour");
			return 1;
	    }
    }
    else
	{
        SPD(playerid, MENU_CREATE_MODEL_OBJECT, DIALOG_STYLE_LIST, "Main Attach", (OMInfo[idstock[playerid]][om_position] == HAND_POSITION_LR) ? ("Main Gauche\nMain Droite") : ("Main Gauche/Droite"), "Continuer", "Retour");
	}
	return 1;
}
public OnPlayerClickPlayer(playerid, clickedplayerid, source)
{
    return 1;
}
public OnPlayerDataLoad(playerid)
{



    /*new rows, fields;
    cache_get_data(rows, fields, mysql_handle);
    if(rows)
    {
        new temp[STRING_3];
        cache_get_row(0, COLONNE_id, temp, mysql_handle); PInfo[playerid][id] = strval(temp);
        cache_get_row(0, COLONNE_token, PInfo[playerid][token], mysql_handle);
        cache_get_row(0, COLONNE_admin, temp, mysql_handle); PInfo[playerid][admin] = strval(temp);
        cache_get_row(0, COLONNE_argent, temp, mysql_handle); PInfo[playerid][argent] = strval(temp);
        cache_get_row(0, COLONNE_banque, temp, mysql_handle); PInfo[playerid][banque] = strval(temp);
        cache_get_row(0, COLONNE_date_inscription, PInfo[playerid][date_inscription], mysql_handle);
        cache_get_row(0, COLONNE_ip_inscription, PInfo[playerid][ip_inscription], mysql_handle);
        cache_get_row(0, COLONNE_heures, temp, mysql_handle); PInfo[playerid][heures] = strval(temp);
        cache_get_row(0, COLONNE_sexe, temp, mysql_handle); PInfo[playerid][sexe] = strval(temp);
        cache_get_row(0, COLONNE_age, temp, mysql_handle); PInfo[playerid][age] = strval(temp);
        cache_get_row(0, COLONNE_origine, temp, mysql_handle); PInfo[playerid][origine] = strval(temp);
        cache_get_row(0, COLONNE_roleplay, temp, mysql_handle); PInfo[playerid][roleplay] = strval(temp);
        cache_get_row(0, COLONNE_activite, temp, mysql_handle); PInfo[playerid][activite] = strval(temp);
        cache_get_row(0, COLONNE_utilisateur, PInfo[playerid][utilisateur], mysql_handle);
        State_On(VarState[playerid], PLAYER_CONNECTED);

        if(strcmp(PInfo[playerid][token],GetPIp(playerid)) != 0)
        {
            SendMessage(playerid,MSG_ERREUR,"Token invalide. Veuillez vous reconnecter depuis l'application. Si le problème persiste, contactez l'administration.");
            KickEx(playerid,"[SERVEUR]","Token Invalide",0);
        }
    }
    else
    {
        SendMessage(playerid,MSG_INFO,"Vous n'avez pas de compte. Veuillez en créer un depuis l'application.");
        KickEx(playerid,"[SERVEUR]","Pas de compte",0);
    }*/
    return 1;
}
/*public OnObjectsDataLoad()
{
    new rows, fields;
    cache_get_data(rows, fields, mysql_handle);
    for(new i=0;i<rows;i++)
    {
        new temp[STRING_3];
        cache_get_row(i, OCOLONNE_id, temp, mysql_handle); OInfo[i][id] = strval(temp);
        cache_get_row(i, OCOLONNE_nom, OInfo[i][nom], mysql_handle);
        cache_get_row(i, OCOLONNE_id_objet, temp, mysql_handle); OInfo[i][id_objet] = strval(temp);
        cache_get_row(i, OCOLONNE_sol_posx, temp, mysql_handle); OInfo[i][sol_posx] = strval(temp);
        cache_get_row(i, OCOLONNE_sol_posy, temp, mysql_handle); OInfo[i][sol_posy] = strval(temp);
        cache_get_row(i, OCOLONNE_sol_posz, temp, mysql_handle); OInfo[i][sol_posz] = strval(temp);
        cache_get_row(i, OCOLONNE_sol_rotx, temp, mysql_handle); OInfo[i][sol_rotx] = strval(temp);
        cache_get_row(i, OCOLONNE_sol_roty, temp, mysql_handle); OInfo[i][sol_roty] = strval(temp);
        cache_get_row(i, OCOLONNE_sol_rotz, temp, mysql_handle); OInfo[i][sol_rotz] = strval(temp);
        cache_get_row(i, OCOLONNE_lhand_posx, temp, mysql_handle); OInfo[i][lhand_posx] = strval(temp);
        cache_get_row(i, OCOLONNE_lhand_posy, temp, mysql_handle); OInfo[i][lhand_posy] = strval(temp);
        cache_get_row(i, OCOLONNE_lhand_posz, temp, mysql_handle); OInfo[i][lhand_posz] = strval(temp);
        cache_get_row(i, OCOLONNE_lhand_rotx, temp, mysql_handle); OInfo[i][lhand_rotx] = strval(temp);
        cache_get_row(i, OCOLONNE_lhand_roty, temp, mysql_handle); OInfo[i][lhand_roty] = strval(temp);
        cache_get_row(i, OCOLONNE_lhand_rotz, temp, mysql_handle); OInfo[i][lhand_rotz] = strval(temp);
        cache_get_row(i, OCOLONNE_rhand_posx, temp, mysql_handle); OInfo[i][rhand_posx] = strval(temp);
        cache_get_row(i, OCOLONNE_rhand_posy, temp, mysql_handle); OInfo[i][rhand_posy] = strval(temp);
        cache_get_row(i, OCOLONNE_rhand_posz, temp, mysql_handle); OInfo[i][rhand_posz] = strval(temp);
        cache_get_row(i, OCOLONNE_rhand_rotx, temp, mysql_handle); OInfo[i][rhand_rotx] = strval(temp);
        cache_get_row(i, OCOLONNE_rhand_roty, temp, mysql_handle); OInfo[i][rhand_roty] = strval(temp);
        cache_get_row(i, OCOLONNE_rhand_rotz, temp, mysql_handle); OInfo[i][rhand_rotz] = strval(temp);
        cache_get_row(i, OCOLONNE_bhand_posx, temp, mysql_handle); OInfo[i][bhand_posx] = strval(temp);
        cache_get_row(i, OCOLONNE_bhand_posy, temp, mysql_handle); OInfo[i][bhand_posy] = strval(temp);
        cache_get_row(i, OCOLONNE_bhand_posz, temp, mysql_handle); OInfo[i][bhand_posz] = strval(temp);
        cache_get_row(i, OCOLONNE_bhand_rotx, temp, mysql_handle); OInfo[i][bhand_rotx] = strval(temp);
        cache_get_row(i, OCOLONNE_bhand_roty, temp, mysql_handle); OInfo[i][bhand_roty] = strval(temp);
        cache_get_row(i, OCOLONNE_bhand_rotz, temp, mysql_handle); OInfo[i][bhand_rotz] = strval(temp);
    }
    printf("Nombres de model véhicules : %d", GetNbrModelVehicle());
    return 1;
}*/
stock KickEx(playerid, auteur[], raison[], silencieux)
{
    SetTimerEx("TimerKick",1000,false,"i",playerid);
    new str[MAX_INPUT];
    if(!silencieux)
    {
        format(str,sizeof(str),"%s a été kické du serveur par "gris_clair"%s"blanc" pour "gris_clair"%s"blanc".",GetPName(playerid),auteur,raison);
        SendMessageToAll(MSG_SERVEUR,str);
        ClearChat(playerid,30);
        format(str,sizeof(str),"Vous avez été kické du serveur par "gris_clair"%s"blanc" pour "gris_clair"%s"blanc".",auteur,raison);
        SendMessage(playerid,MSG_SERVEUR,str);
    }
    else
    {
        format(str,sizeof(str),"Vous avez été kické du serveur par "gris_clair"%s"blanc" pour "gris_clair"%s"blanc".",auteur,raison);
        SendMessage(playerid,MSG_SERVEUR,str);
    }
    //Log du kick avec admin et raison
    return 1;
}
public TimerKick(playerid)
{
    Kick(playerid);
}
/*stock ChargementListeObjets()
{
    mysql_function_query(mysql_handle, "SELECT * FROM `objets`", true, "OnObjectsDataLoad", "");
}*/
stock LoadPlayerTextures(playerid)
{
    #pragma unused playerid
    return 0;
}
stock SetPlayerSpawn(playerid)
{
    //new Float:PosX[MAX_PLAYERS], Float:PosY[MAX_PLAYERS], Float:PosZ[MAX_PLAYERS], Float:PosA[MAX_PLAYERS];
    if(PInfo[playerid][activite] != 0)
    {

    }
    /*SetPlayerPos(playerid,-183.0428,1133.5691,19.7422);
    SetPlayerFacingAngle(playerid,90.2917);

    SetPlayerCameraPos(playerid, -183.0428,1133.5691,19.7422+50);
    SetPlayerCameraLookAt(playerid, -183.0428,1133.5691,19.7422); Début d'idée d'un chargement du personnage comme gta v. */

    SetSpawnInfo(playerid, 0,217,-183.0428,1133.5691,19.7422,90.2917,0,0,0,0,0,0);
    SpawnPlayer(playerid);
    varTimer1s[playerid] = SetTimerEx("pTimer1s", 1000, true, "i", playerid);
    return 1;
}
stock ShowPlayerHelpDialog(playerid,menu)
{
    switch(menu)
    {
        case 0: SPD(playerid,DIALOG_AIDE_MAIN,DIALOG_STYLE_LIST,"Aide","Commandes générales","Valider","Fermer");
        case 1: SPD(playerid,DIALOG_AIDE_GENERAL,DIALOG_STYLE_MSGBOX,"Commandes générales","COMMANDES :\n\nTchat : /me - /do - /crier - /chu(choter) - /local - /b\nCompte : /stats","Retour","Fermer");
    }
}


// PHYSICS
/*public PHY_OnObjectUpdate(objectid)
{
   //printf("Update Object, obj : %d, move : %d, playerid : %d, player obj : %d", objectid, PHY_IsObjectLaunch(objectid), obj_playerid_launch[objectid], PLaunchObject[obj_playerid_launch[objectid]][HAND_RIGHT]);
   return 1;
}
public PHY_OnObjectStop(objectid)
{
   printf("Stop obj : %d");
   if(objectid == PLaunchObject[obj_playerid_launch[objectid]][HAND_RIGHT])
   {
	   new Float:pos[3];
       OInfo[PInfo[obj_playerid_launch[objectid]][righthand]][o_playerid] = -1;
       GetObjectPos(objectid, arr3{pos});
       //GetObjectRot(objectid, arr3{rot});
       OInfo[PInfo[obj_playerid_launch[objectid]][righthand]][o_sol_posx] = pos[0];
       OInfo[PInfo[obj_playerid_launch[objectid]][righthand]][o_sol_posy] = pos[1];
       OInfo[PInfo[obj_playerid_launch[objectid]][righthand]][o_sol_posz] = pos[2] + OInfo[PInfo[obj_playerid_launch[objectid]][righthand]][o_sol_reglagez];
       O_Object[PInfo[obj_playerid_launch[objectid]][righthand]] = CreateDynamicObject(OInfo[PInfo[obj_playerid_launch[objectid]][righthand]][o_idobjet],
	   arr3{pos},
	   OInfo[PInfo[obj_playerid_launch[objectid]][righthand]][o_sol_rotx],
	   OInfo[PInfo[obj_playerid_launch[objectid]][righthand]][o_sol_roty],
	   OInfo[PInfo[obj_playerid_launch[objectid]][righthand]][o_sol_rotz],
	   OInfo[PInfo[obj_playerid_launch[objectid]][righthand]][o_virtuel],OInfo[PInfo[obj_playerid_launch[objectid]][righthand]][o_interieur]);
       PInfo[obj_playerid_launch[objectid]][righthand] = 0;
       DestroyObject(PLaunchObject[obj_playerid_launch[objectid]][HAND_RIGHT]);
       obj_playerid_launch[objectid] = 0;
       PLaunchObject[obj_playerid_launch[objectid]][HAND_RIGHT] = 0;
       return 1;
   }
   return 1;
}*/



/*// SAMP +

public OnPlayerOpenPauseMenu(playerid)
{
	print("menu pause open");
}
public OnPlayerClosePauseMenu(playerid)
{
	print("menu pause close");
}
public OnPlayerEnterPauseSubmenu(playerid, from, to)
{
    printf("1- %d, %d %d",playerid, from, to);
	if(to == PAUSE_ID_MAP)
	{
		printf("true %d",playerid);
        TogglePauseMenuAbility(playerid, true);
	}
}*/
