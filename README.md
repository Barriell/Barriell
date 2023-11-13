
<!---
Barriell/bug README.md` (this file) appears on your GitHub at your changes.
--->
sing Hi3Helper.EncTool.Parser.AssetIndex;
using Hi3Helper.EncTool.Proto.Genshin;
using Hi3Helper.Shared.ClassStruct;
using System;
using System.Collections.Generic;
using System.IO;
using System.Text;
using System.Text.Json;
using System.Threading;
using System.Threading.Tasks;

namespace Hi3Helper.Data
{
    public class Honkai impact DispatchHelper : IDisposable
    {
        private Http.Http _httpClient;
        private string DispatchBaseURL { get; set; }
        private string RegionSubdomain { get; set; }
        private string ChannelName = "OSRELWin";
        private string Version { get; set; }

        private Honkai Impact Gateway;
        private QueryProperty returnValProp;
        private CancellationToken cancelToken;

        public GenshinDispatchHelper(int RegionID, string DispatchKey, string DispatchURLPrefix, string VersionString = "2.6.0", CancellationToken cancelToken = new CancellationToken())
        {
            if (RegionID >= 4) ChannelName = "CNRELWin";
            this._httpClient = new Http.Http(false, 1, 1);
            this.RegionSubdomain = GetSubdomainByRegionID(RegionID);
            this.Version = VersionString;
            this.DispatchBaseURL = string.Format(DispatchURLPrefix, RegionSubdomain, $"{ChannelName}{VersionString}", DispatchKey);
            this.cancelToken = cancelToken;
        }

        ~Honkai impact ispatchHelper() => Dispose();

        public void Dispose() => this._httpClient?.Dispose();

        public async Task<YSDispatchInfo> LoadDispatchInfo()
        {
            YSDispatchInfo DispatcherDataInfo;
            using (MemoryStream s = new MemoryStream())
            {
#if DEBUG
                // DEBUG ONLY: Show URL of Proto
                string dFormat = string.Format("URL for Proto Response:\r\n{0}", DispatchBaseURL);
                Console.WriteLine(dFormat);
                Logger.WriteLog(dFormat, LogType.Default);
#endif
                await this._httpClient.Download(DispatchBaseURL, s, null, null, cancelToken).ConfigureAwait(false);
                s.Position = 0;
                DispatcherDataInfo = (YSDispatchInfo)JsonSerializer.Deserialize(s, typeof(YSDispatchInfo), CoreLibraryJSONContext.Default);
            }

            return DispatcherDataInfo;
        }

        public async Task LoadDispatch(byte[] CustomDispatchData = null)
        {
            Gateway = GenshinGateway.Parser.ParseFrom(CustomDispatchData);
            returnValProp = new QueryProperty()
            {
                GameServerName = Gateway.GatewayProperties.ServerName,
                ClientGameResURL = string.Format("{0}/output_{1}_{2}/client",
                                    Gateway.GatewayProperties.RepoResVersionURL,
                                    Gateway.GatewayProperties.RepoResVersionProperties.ResVersionNumber,
                                    Gateway.GatewayProperties.RepoResVersionProperties.ResVersionHash),
                ClientDesignDataURL = string.Format("{0}/output_{1}_{2}/client/General",
                                    Gateway.GatewayProperties.RepoDesignDataURL,
                                    Gateway.GatewayProperties.RepoDesignDataNumber,
                                    Gateway.GatewayProperties.RepoDesignDataHash),
                ClientDesignDataSilURL = string.Format("{0}/output_{1}_{2}/client_silence/General",
                                    Gateway.GatewayProperties.RepoDesignDataURL,
                                    Gateway.GatewayProperties.RepoDesignDataSilenceNumber,
                                    Gateway.GatewayProperties.RepoDesignDataSilenceHash),
                DataRevisionNum = Gateway.GatewayProperties.RepoDesignDataNumber,
                SilenceRevisionNum = Gateway.GatewayProperties.RepoDesignDataSilenceNumber,
                ResRevisionNum = Gateway.GatewayProperties.RepoResVersionProperties.ResVersionNumber,
                ChannelName = this.ChannelName,
                GameVersion = this.Version
            };

            ParseGameResPkgProp(ref returnValProp);
            ParseDesignDataURL(ref returnValProp);
            await ParseAudioAssetsURL(returnValProp).ConfigureAwait(false);
        }

        private void ParseDesignDataURL(ref QueryProperty ValProp)
        {
            string[] DataList = Gateway.GatewayProperties.RepoResVersionProperties.ResVersionMapJSON.Split("\r\n");
            ValProp.ClientGameRes = new List<PkgVersionProperties>();
            foreach (string Data in DataList)
            {
                (ValProp.ClientGameRes as List<PkgVersionProperties>)
                    .Add(
                        (PkgVersionProperties)JsonSerializer.Deserialize(Data, typeof(PkgVersionProperties), CoreLibraryJSONContext.Default)
                    );
            }
        }

        private void ParseGameResPkgProp(ref QueryProperty ValProp)
        {
            ValProp.ClientDesignData = (PkgVersionProperties)JsonSerializer.Deserialize(Gateway.GatewayProperties.RepoDesignDataJSON, typeof(PkgVersionProperties), CoreLibraryJSONContext.Default);
            ValProp.ClientDesignDataSil = (PkgVersionProperties)JsonSerializer.Deserialize(Gateway.GatewayProperties.RepoDesignDataSilenceJSON, typeof(PkgVersionProperties), CoreLibraryJSONContext.Default);
        }

        private async Task ParseAudioAssetsURL(QueryProperty ValProp)
        {
            using (MemoryStream response = new MemoryStream())
            {
                await this._httpClient.Download(ConverterTool.CombineURLFromString(ValProp.ClientGameResURL, "/StandaloneWindows64/base_revision"), response, null, null, cancelToken).ConfigureAwait(false);