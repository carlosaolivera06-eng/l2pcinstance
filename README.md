	private static L2PcInstance restore(int objectId)
	{
		L2PcInstance player = null;
		double curHp = 0;
		double curCp = 0;
		double curMp = 0;
		
		Connection con = null;
		try
		{
			con = L2DatabaseFactory.getInstance().getConnection();
			PreparedStatement statement = con.prepareStatement(RESTORE_CHARACTER);
			
			statement.setInt(1, objectId);
			ResultSet rset = statement.executeQuery();
			
			while (rset.next())
			{
				final int activeClassId = rset.getInt("classid");
				final boolean female = rset.getInt("sex") != 0;
				final L2PcTemplate template = CharTemplateTable.getInstance().getTemplate(activeClassId);
				PcAppearance app = new PcAppearance(rset.getByte("face"), rset.getByte("hairColor"), rset.getByte("hairStyle"), female);
				
				player = new L2PcInstance(objectId, template, rset.getString("account_name"), app);
				
				player.restorePremServiceData(player, rset.getString("account_name"));
				
				if (Config.RON_CUSTOM)
				{
					player.restoreBuffSlotsData(player, rset.getString("account_name"));
					player.restoreHourBuffsData(player, rset.getString("account_name"));
				}
				
				player.setName(rset.getString("char_name"));
				player._lastAccess = rset.getLong("lastAccess");
				
				player.getStat().setExp(rset.getLong("exp"));
				player.setExpBeforeDeath(rset.getLong("expBeforeDeath"));
				player.getStat().setLevel(rset.getByte("level"));
				player.getStat().setSp(rset.getInt("sp"));
				
				player.setWantsPeace(rset.getInt("wantspeace"));
				
				player.setHeading(rset.getInt("heading"));
				
				player.setKarma(rset.getInt("karma"));
				player.setPvpKills(rset.getInt("pvpkills"));
				player.setPkKills(rset.getInt("pkkills"));
				player.setOnlineTime(rset.getLong("onlinetime"));
				player.setNewbie(rset.getInt("newbie") == 1);
				player.setNoble(rset.getInt("nobless") == 1);
				player.setClanJoinExpiryTime(rset.getLong("clan_join_expiry_time"));
				player.setFirstLog(rset.getInt("first_log"));
				player.pcBangPoint = rset.getInt("pc_point");
				
				if (player.getClanJoinExpiryTime() < System.currentTimeMillis())
				{
					player.setClanJoinExpiryTime(0);
				}
				player.setClanCreateExpiryTime(rset.getLong("clan_create_expiry_time"));
				if (player.getClanCreateExpiryTime() < System.currentTimeMillis())
				{
					player.setClanCreateExpiryTime(0);
				}
				
				int clanId = rset.getInt("clanid");
				player.setPowerGrade((int) rset.getLong("power_grade"));
				player.setPledgeType(rset.getInt("subpledge"));
				player.setLastRecomUpdate(rset.getLong("last_recom_date"));
				
				if (clanId > 0)
				{
					player.setClan(ClanTable.getInstance().getClan(clanId));
				}
				
				if (player.getClan() != null)
				{
					if (player.getClan().getLeaderId() != player.getObjectId())
					{
						if (player.getPowerGrade() == 0)
						{
							player.setPowerGrade(6);
						}
						player.setClanPrivileges(player.getClan().getRankPrivs(player.getPowerGrade()));
					}
					else
					{
						player.setClanPrivileges(L2Clan.CP_ALL);
						player.setPowerGrade(1);
					}
				}
				else
				{
					player.setClanPrivileges(L2Clan.CP_NOTHING);
				}
				
				player.setDeleteTimer(rset.getLong("deletetime"));
				
				player.setTitle(rset.getString("title"));
				player.setAccessLevel(0);
				player.setFistsWeaponItem(player.findFistsWeaponItem(activeClassId));
				player.setUptime(System.currentTimeMillis());
				
				curHp = rset.getDouble("curHp");
				curCp = rset.getDouble("curCp");
				curMp = rset.getDouble("curMp");
				
				player.checkRecom(rset.getInt("rec_have"), rset.getInt("rec_left"));
				
				player._classIndex = 0;
				try
				{
					player.setBaseClass(rset.getInt("base_class"));
				}
				catch (Exception e)
				{
					if (Config.ENABLE_ALL_EXCEPTIONS)
					{
						e.printStackTrace();
					}
					
					player.setBaseClass(activeClassId);
				}
				
				if (restoreSubClassData(player))
				{
					if (activeClassId != player.getBaseClass())
					{
						for (SubClass subClass : player.getSubClasses().values())
						{
							if (subClass.getClassId() == activeClassId)
							{
								player._classIndex = subClass.getClassIndex();
							}
						}
					}
				}
				if (player.getClassIndex() == 0 && activeClassId != player.getBaseClass())
				{
					// Subclass in use but doesn't exist in DB -
					// a possible restart-while-modifysubclass cheat has been attempted.
					// Switching to use base class
					player.setClassId(player.getBaseClass());
					LOG.warn("Player " + player.getName() + " reverted to base class. Possibly has tried a relogin exploit while subclassing.");
				}
				else
				{
					player._activeClass = activeClassId;
				}
				
				player.setApprentice(rset.getInt("apprentice"));
				player.setSponsor(rset.getInt("sponsor"));
				player.setLvlJoinedAcademy(rset.getInt("lvl_joined_academy"));
				player.setIsIn7sDungeon(rset.getInt("isin7sdungeon") == 1 ? true : false);
				
				player.setPunishLevel(rset.getInt("punish_level"));
				if (player.getPunishLevel() != PunishLevel.NONE)
				{
					player.setPunishTimer(rset.getLong("punish_timer"));
				}
				else
				{
					player.setPunishTimer(0);
				}
				
				try
				{
					player.getAppearance().setNameColor(Integer.decode(new StringBuilder().append("0x").append(rset.getString("name_color")).toString()).intValue());
					player.getAppearance().setTitleColor(Integer.decode(new StringBuilder().append("0x").append(rset.getString("title_color")).toString()).intValue());
				}
				catch (Exception e)
				{
					if (Config.ENABLE_ALL_EXCEPTIONS)
					{
						e.printStackTrace();
					}
				}
				
				CursedWeaponsManager.getInstance().checkPlayer(player);
				player.setAllianceWithVarkaKetra(rset.getInt("varka_ketra_ally"));
				
				player.setDeathPenaltyBuffLevel(rset.getInt("death_penalty_level"));
				
				player.setAutoLootEnabled(rset.getInt("autoloot") == 1 ? true : false);
				player.setAutoLootHerbs(rset.getInt("autoloot_herbs") == 1 ? true : false);
				
				player.setBlockAllBuffs(rset.getInt("blockbuff") == 1 ? true : false);
				player.setExpOn(rset.getInt("gainexp") == 1 ? true : false);
				player.setTitleOn(rset.getInt("titlestatus") == 1 ? true : false);
				player.setMessageRefusal(rset.getInt("pm") == 1 ? true : false);
				player.setTradeRefusal(rset.getInt("trade") == 1 ? true : false);
				player.setScreentxt(rset.getInt("screentxt") == 1 ? true : false);
				player.setTeleport(rset.getInt("teleport") == 1 ? true : false);
				player.setEffects(rset.getInt("effects") == 1 ? true : false);
				
				// Cargamos el skin
				int skinRace = rset.getInt("custom_race_skin");
				player.setCustomRaceSkin(skinRace);
				player.setCustomClassSkin(rset.getInt("custom_class_skin"));
				
				// --- PARCHE DE COLISIÓN, CLASE Y SEXO AL LOGUEAR ---
				if (skinRace != -1)
				{
					int claseReal = player.getClassId().getId();
					int tId = 0;
					switch (skinRace)
					{
						case 0:
							tId = 0;
							break;
						case 1:
							tId = 18;
							break;
						case 2:
							tId = 31;
							break;
						case 3:
							tId = 44;
							break;
						case 4:
							tId = 53;
							break;
					}
					
					player.setTemplate(CharTemplateTable.getInstance().getTemplate(tId));
					player.setClassId(claseReal);
					
					// Forzamos al objeto Appearance para que el Lobby mande los datos correctos
					// Esto evita que aparezcas como mujer o con la raza base en la selección de PJ
					player.getAppearance().setSex(false);
				}
				
				// Set the x,y,z position of the L2PcInstance and make it invisible
				player.setXYZInvisible(rset.getInt("x"), rset.getInt("y"), rset.getInt("z"));
				
				// Retrieve the name and ID of the other characters assigned to this account.
				PreparedStatement stmt = con.prepareStatement("SELECT obj_Id, char_name FROM characters WHERE account_name=? AND obj_Id<>?");
				stmt.setString(1, player._accountName);
				stmt.setInt(2, objectId);
				ResultSet chars = stmt.executeQuery();
				
				while (chars.next())
				{
					Integer charId = chars.getInt("obj_Id");
					String charName = chars.getString("char_name");
					player._chars.put(charId, charName);
				}
				
				chars.close();
				stmt.close();
				break;
			}
			
			rset.close();
			statement.close();
		}
		catch (Exception e)
		{
			LOG.error("Could not restore char data:");
			e.printStackTrace();
		}
		finally
		{
			CloseUtil.close(con);
		}
		
		if (player != null)
		{
			player.restoreCharData();
			player.rewardSkills(true);
			
			player.setPet(L2World.getInstance().getPet(player.getObjectId()));
			if (player.getPet() != null)
			{
				player.getPet().setOwner(player);
			}
			
			player.refreshOverloaded();
			
			player.restoreFriendList();
			player.restoreDressMeData();
			
			player.setCurrentHpDirect(curHp);
			player.setCurrentCpDirect(curCp);
			player.setCurrentMpDirect(curMp);
		}
		    
		return player;
	}
