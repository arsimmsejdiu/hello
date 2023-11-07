import {BASE, S3_URL_NEWS} from "@assets/constant";
import {Close} from "@assets/index";
import {ImageAtom} from "@components/Atom";
import {PaginationBar} from "@components/PaginationBar";
import {useAppSelector} from "@hooks/index";
import {NewsPhotoName} from "@models/data/NewsPhotoName";
import {useFocusEffect} from "@react-navigation/native";
import {loadData, storeData} from "@services/asyncStorage";
import {HomeStackScreenProps} from "@stacks/types";
import {newsPopupStyles} from "@styles/NewsPopupStyles";
import React, {FC, memo, ReactElement, useCallback, useState} from "react";
import {useTranslation} from "react-i18next";
import {ImageBackground, StatusBar, TouchableHighlight, TouchableOpacity, View, ViewProps} from "react-native";
import {useSharedValue} from "react-native-reanimated";
import Carousel from "react-native-reanimated-carousel";
import {useSafeAreaInsets} from "react-native-safe-area-context";

import MyConfig from "../../config";

const {APP_ENV} = MyConfig;


type Props = ViewProps & HomeStackScreenProps<"News">;

const NewsScreen: FC<Props> = ({navigation}): ReactElement => {
    const insets = useSafeAreaInsets();
    const newsPhotoNames = useAppSelector(state => state.initialState.newsPhotoNames);
    const newsActive = useAppSelector(state => state.initialState.newsIsActive);
    const userCityId = useAppSelector(state => state.auth.me?.city_id);
    const [locale, setLocale] = useState("fr");
    const {i18n} = useTranslation();

    const [newsPhotoFiltered, setNewsPhotoFiltered] = useState<NewsPhotoName[]>([]);
    const progressValue = useSharedValue<number>(0);
    const [isMoreThan24h, setIsMoreThan24h] = useState(false);

    const [filterDone, setFilterDone] = useState(false);
    const [checkValidityDone, setCheckValidityDone] = useState(false);
    const [show, setShow] = useState(false);


    const showNewsPopup = async (): Promise<boolean> => {
        const alreadySeen = await loadData("@NewsViewedData");
        let alreadySeenBool = !(typeof alreadySeen === "undefined" || alreadySeen === null);
        if (alreadySeenBool) {
            const alreadySeenNumber = Number(alreadySeen);
            if (Number.isNaN(alreadySeenNumber)) {
                alreadySeenBool = false;
            } else {
                const currentTimeStamp = Date.now();
                const differenceInMilliseconds = currentTimeStamp - alreadySeenNumber;
                const differenceInHours = differenceInMilliseconds / (1000 * 60 * 60);
                alreadySeenBool = differenceInHours < 24;
            }
        }

        if (APP_ENV === "production") {
            await storeData("@NewsViewedData", Date.now() + "");
        }
        return !alreadySeenBool;
    };

    const handleCloseModal = () => {
        console.log("NewsScreen - Closing modal, navigate to map");
        navigation.navigate("Map");
    };

    useFocusEffect(useCallback(() => {
        const newLocale = i18n.language === "fr" ? "fr" : "en";
        setLocale(newLocale);
    }, [i18n.language]));

    const filterNewsPhoto = (newsPhoto: NewsPhotoName, userCityId: number | null | undefined) => {
        if (typeof newsPhoto.cities_ids !== "undefined" && !newsPhoto.cities_ids.includes(userCityId ?? 0)) {
            return false;
        }
        return true;
    };

    useFocusEffect(useCallback(() => {
        console.log("NewsScreen - Filter images: ");
        console.log("NewsScreen - newsPhotoNames.length = ", newsPhotoNames.length);
        console.log("NewsScreen - userCityId = ", userCityId);
        const filtered = newsPhotoNames.filter((n) => filterNewsPhoto(n, userCityId));
        setNewsPhotoFiltered(filtered);
        setFilterDone(true);
        return () => {
            console.log("NewsScreen - in return of filter, set filterDone back to false");
            setFilterDone(false);
        };
    }, [newsPhotoNames, userCityId]));

    const handleRedirectTo = (redirectTo?: string | null | undefined) => {
        if (typeof redirectTo !== "undefined" && redirectTo !== null) {
            console.log("NewsScreen - image pressed, redirect to ", redirectTo);
            //Ignored because navigation is strongly typed and generic type string is not allowed here
            // eslint-disable-next-line @typescript-eslint/ban-ts-comment
            // @ts-ignore
            navigation.navigate(redirectTo);
        }
    };

    useFocusEffect(useCallback(() => {
        showNewsPopup().then(result => {
            setIsMoreThan24h(result);
            setCheckValidityDone(true);
        });
        return () => {
            console.log("NewsScreen - Loosing focus, set show back to false");
            setCheckValidityDone(false);
            setShow(false);
        };
    }, []));

    useFocusEffect(useCallback(() => {
        console.log("NewsScreen - filterDone = ", filterDone);
        console.log("NewsScreen - checkValidityDone = ", checkValidityDone);
        if (filterDone && checkValidityDone) {
            console.log("NewsScreen - newsActive = ", newsActive);
            console.log("NewsScreen - isMoreThan24h = ", isMoreThan24h);
            if (!newsActive || !isMoreThan24h) {
                console.log("NewsScreen - it's not time to show news, redirecting to map");
                navigation.navigate("Map");
            } else if ((newsPhotoFiltered?.length ?? 0) === 0) {
                console.log("NewsScreen - filtered is empty, navigating to map ");
                navigation.navigate("Map");
            } else {
                console.log("NewsScreen - All set up, showing news");
                setShow(true);
            }
        }
    }, [filterDone, checkValidityDone, show, newsActive, newsPhotoFiltered]));

    if(!show){
        return <View></View>;
    }

    return (
        <View style={{
            ...newsPopupStyles.container,
        }}>
            <StatusBar translucent backgroundColor="transparent" barStyle={"dark-content"}/>
            <Carousel
                data={newsPhotoFiltered}
                width={BASE.window.width}
                height={BASE.window.screenHeight}
                onProgressChange={(_, absoluteProgress) =>
                    (progressValue.value = absoluteProgress)
                }
                renderItem={({item, index}) => {
                    return (
                        <TouchableHighlight
                            touchSoundDisabled={false}
                            onPress={() => handleRedirectTo(item.redirect_to)}
                            activeOpacity={1}
                            key={index}
                            style={newsPopupStyles.item}
                        >
                            <ImageBackground
                                source={{uri: `${S3_URL_NEWS}${locale === "fr" ? item.names.fr : item.names.en}`}}
                                style={{
                                    flex: 1,
                                    width: "100%",
                                    height: "100%",
                                }}
                                resizeMode="cover"
                            />
                        </TouchableHighlight>
                    );
                }}
            />
            <TouchableOpacity
                onPress={() => handleCloseModal()}
                style={{...newsPopupStyles.imageClose, top: newsPopupStyles.imageClose.top + insets.top}}
            >
                <ImageAtom
                    source={Close}
                    style={newsPopupStyles.imageSize}
                    resizeMode={"cover"}
                />
            </TouchableOpacity>
            <View style={{
                ...newsPopupStyles.paginationBarContainer,
                bottom: newsPopupStyles.paginationBarContainer.bottom + insets.bottom
            }}>
                {newsPhotoFiltered.map((page, index) => {
                    return (
                        <PaginationBar
                            animValue={progressValue}
                            index={index}
                            key={index}
                            length={newsPhotoFiltered.length}
                        />
                    );
                })}
            </View>
        </View>
    );
};

export default memo(NewsScreen);
