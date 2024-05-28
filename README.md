# Temp

 *********dCom************************ 
Configuration:
    ConfigItem:
        ConfigItem:
            if (configurationParameters[10].Equals("#"))
            {
                            scalingFactor = 1;
            }
            else
            {
                            Double.TryParse(configurationParameters[10], out doubleTemp);
                            scalingFactor = doubleTemp;
            }


            if (configurationParameters[11].Equals("#"))
            {
                            deviation = 0;
            }
            else
            {
                            Double.TryParse(configurationParameters[11], out doubleTemp);
                            deviation = doubleTemp;
            }

            if (configurationParameters[12].Equals("#"))
            {
                            EGU_Max = 1;
            }
            else
            {
                            Double.TryParse(configurationParameters[12], out doubleTemp);
                            EGU_Max = doubleTemp;
            }

            if (configurationParameters[13].Equals("#"))
            {
                            EGU_Min = 0;
            }
            else
            {
                            Double.TryParse(configurationParameters[13], out doubleTemp);
                            EGU_Min = doubleTemp;
            }

            if (configurationParameters[14].Equals("#"))
            {
                            abnormalValue = 0;
            }
            else
            {
                            Int32.TryParse(configurationParameters[14], out temp);
                            abnormalValue = (ushort)temp;
            }

            if (configurationParameters[15].Equals("#"))
            {
                            lowLimit = 0;
            }
            else
            {
                            Double.TryParse(configurationParameters[15], out doubleTemp);
                            lowLimit = doubleTemp;
            }
            if (configurationParameters[16].Equals("#"))
            {
                            highLimit = 1;
            }
            else
            {
                            Double.TryParse(configurationParameters[16], out doubleTemp);
                            highLimit = doubleTemp;
            }
 



*********ProcessingModule************************ 
 
    AlarmProcessor:
        -GetAlarmForAnalogPoint:
            ************************************
            if (configItem.HighLimit < eguValue)
            {
                return AlarmType.HIGH_ALARM;
            }
            else if (configItem.LowLimit > eguValue)
            {
                return AlarmType.LOW_ALARM;
            }
            else if (configItem.EGU_Max < eguValue || configItem.EGU_Min > eguValue)
            {
                return AlarmType.REASONABILITY_FAILURE;
            }
            ************************************

        -GetAlarmForDigitalPoint:
            ************************************
            if (state == configItem.AbnormalValue)
            {
                return AlarmType.ABNORMAL_VALUE;
            }

    AutomationManager:
        -AutomationWorker_DoWork:
            EGUConverter conv = new EGUConverter();
            while (!disposedValue)
            {
                List<PointIdentifier> pointList = new List<PointIdentifier>();

                //Udaljenost kapije MAX-700(Ne pise koliko je max ja stavio) na [0] mestu u listi
                pointList.Add(new PointIdentifier(PointType.ANALOG_OUTPUT, 1000));

                //Indikacija prepreke na [1] mestu u listi
                pointList.Add(new PointIdentifier(PointType.DIGITAL_INPUT, 2000));

                //Open na [2] mestu u listi
                //Ako je ukljucen oduzimamo -10 na vrednost(Udaljenost kapije) svake sekunde Otvaramo Kapiju
                pointList.Add(new PointIdentifier(PointType.DIGITAL_OUTPUT, 3000));

                //Close na [3] mestu u listi
                //Ako je ukljucen dodajemo +10 na vrednost(Udaljenost kapije) svake sekunde Zatvaramo Kapiju
                pointList.Add(new PointIdentifier(PointType.DIGITAL_OUTPUT, 3001));

                List<IPoint> points = storage.GetPoints(pointList);
                ushort value = points[0].RawValue;

                //Ako je ukljuceno otvaranje
                if (points[2].RawValue == 1)
                {
                    //Otvaraj kapiju smanjuj vrednost za 10cm svake sekunde
                    value -= conv.ConvertToRaw(points[0].ConfigItem.ScaleFactor, points[0].ConfigItem.Deviation, 10);

                    //Ako je kapija dosla do LowLimita = 20cm onda treba da se ugasi otvaranje
                    if (value < points[0].ConfigItem.LowLimit)
                    {
                        //Ugasimo Otvaranje ,Prikazujemo na simulatoru
                        processingManager.ExecuteWriteCommand(points[2].ConfigItem, configuration.GetTransactionId(), configuration.UnitAddress, 3000, 0);
                    }
                    else
                    {
                        //Prikazujemo na simulatoru oduzetu vrednost
                        processingManager.ExecuteWriteCommand(points[0].ConfigItem, configuration.GetTransactionId(), configuration.UnitAddress, 1000, value);
                    }
                                
                                
                }


                if (points[3].RawValue == 1)
                {
                    //Ako postoji prepreka
                    if (points[1].RawValue == 1)
                    {
                        //Otvarati kapiju do LowAlarma
                        value -= conv.ConvertToRaw(points[0].ConfigItem.ScaleFactor, points[0].ConfigItem.Deviation, 10);
                    }
                    else
                    {
                        //Zatvaramo kapiju povecavamo vrednost za 10cm svake sekunde
                        value += conv.ConvertToRaw(points[0].ConfigItem.ScaleFactor, points[0].ConfigItem.Deviation, 10);
                    }


                    //Ako je kapija dosla do HighLimita = 600cm onda treba da se ugasi zatvaranje || Ako je kapija dosla do LowLimita = 20cm onda treba da se ugasi zatvaranje
                    if (value > points[0].ConfigItem.HighLimit || value < points[0].ConfigItem.LowLimit)
                    {
                        //Ugasimo Zatvaranje ,Prikazujemo na simulatoru
                        processingManager.ExecuteWriteCommand(points[3].ConfigItem, configuration.GetTransactionId(), configuration.UnitAddress, 3001, 0);
                    }
                    else
                    {
                        //Prikazujemo na simulatoru sabranu vrednost
                        processingManager.ExecuteWriteCommand(points[0].ConfigItem, configuration.GetTransactionId(), configuration.UnitAddress, 1000, value);
                    }


                }

                automationTrigger.WaitOne(delayBetweenCommands);

    EGUConverter:
        -ConvertToEGU:
            ************************************
            return rawValue * scalingFactor + deviation;
            ************************************
        
        -ConvertToRaw:
            ************************************
            return (ushort)((eguValue - deviation) / scalingFactor);
            ************************************

        
    ProcessingManager:
        -ProcessDigitalPoint:
            point.Alarm = alarmProcessor.GetAlarmForDigitalPoint(newValue, point.ConfigItem);
        
        -ProcessAnalogPoint:
            point.EguValue = eguConverter.ConvertToEGU(point.ConfigItem.ScaleFactor, point.ConfigItem.Deviation, point.RawValue);
            point.Alarm = alarmProcessor.GetAlarmForAnalogPoint(point.EguValue, point.ConfigItem);

    RtuCfg.txt izmeniti

        0 Tip Modbus varijable (DO_REG, DI_REG, IN_REG, HR_INT)
        1 Broj registara (var u bloku)
        2 Adresa (prve var u bloku)
        3 UVIJEK 0
        4 Min vr var
        5 Max vr var
        6 Pocetna vr var
        7 Tip ulaza i izlaza (DO, DI, AI, AO)
        8 Opis var (@DigOut1, @DigOut2, @DigIn1, @AnaIn1, @AnaOut1 i sl.)
        9 Period akv 
        10 faktor skaliranja
        11 odstupanje
        12 EGU_MIN
        13 EGU_MAX
        14 abnormal value
        15 low limit
        16 high limit

    NAPOMENE:
        
Ako signal nema vr nekog parametra napisati #
Digitalni signali:
nemaju faktor skaliranja (10)
nemaju odstupanje (11)
nemaju low limit (15)
nemaju high limit (16)

Ne moraju imati abnormal value (14)
